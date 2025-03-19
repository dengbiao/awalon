# Task 2.2.1: 实现队长轮换与组队机制

## 描述

实现阿瓦隆游戏中的队长轮换和组队机制，包括队长选择、队员选择和队伍验证等功能。

## 详细需求

### 功能需求

1. 队长轮换机制：

   - 游戏开始时随机选择第一位队长
   - 之后按照玩家顺序轮换队长
   - 每次投票拒绝后，队长轮换到下一位玩家
   - 任务完成后，队长轮换到下一位玩家

2. 组队机制：

   - 队长可以选择指定数量的队员（包括自己）
   - 队员数量根据任务编号和玩家人数确定
   - 队长可以修改选择，直到确认队伍
   - 提供队伍历史记录查看功能

3. 队伍验证：
   - 验证队员数量是否符合要求
   - 验证队员是否都是游戏中的玩家
   - 验证是否有重复队员
   - 提供清晰的错误提示

### 界面要求

1. 队长标识：

   - 当前队长头像周围显示明显的队长标识（如皇冠图标）
   - 队长选择队员时有特殊的视觉提示

2. 队员选择界面：

   - 显示所有玩家头像和名称
   - 可选择的玩家有明显的可交互提示
   - 已选择的队员有明显的选中状态
   - 显示当前已选择队员数量和所需队员数量

3. 队伍确认界面：
   - 显示已选择的队员列表
   - 提供修改和确认按钮
   - 确认前有最终确认提示

### 技术要求

1. 实现队长轮换算法，确保公平性
2. 实现队伍验证逻辑，防止无效队伍
3. 实现队伍历史记录功能，支持查看之前的组队情况
4. 确保界面响应迅速，无明显延迟
5. 支持断线重连后恢复当前组队状态

## 实现步骤

1. 设计队长轮换和组队数据结构
2. 实现队长选择算法
3. 实现队员选择界面
4. 实现队伍验证逻辑
5. 实现队伍确认流程
6. 实现队伍历史记录功能
7. 进行测试和优化

## 代码示例

### 队长轮换逻辑

```typescript
// 选择下一位队长
function selectNextCaptain(game: Game): Player {
  // 获取当前队长索引
  const currentCaptainIndex = game.players.findIndex(
    (p) => p.id === game.currentMission.currentCaptainId
  );

  // 计算下一位队长索引
  const nextCaptainIndex = (currentCaptainIndex + 1) % game.players.length;

  // 更新队长信息
  const nextCaptain = game.players[nextCaptainIndex];
  game.currentMission.currentCaptainId = nextCaptain.id;

  // 记录队长历史
  game.captainHistory.push({
    playerId: nextCaptain.id,
    roundNumber: game.currentMission.currentRound,
    missionNumber: game.currentMission.missionNumber,
  });

  return nextCaptain;
}
```

### 队伍验证逻辑

```typescript
// 验证队伍是否有效
function validateTeam(game: Game, memberIds: string[]): boolean {
  // 获取当前任务配置
  const missionConfig = getMissionConfig(
    game.currentMission.missionNumber,
    game.players.length
  );

  // 验证队员数量
  if (memberIds.length !== missionConfig.requiredTeamSize) {
    throw new Error(`队伍必须包含 ${missionConfig.requiredTeamSize} 名队员`);
  }

  // 验证队员是否都是游戏中的玩家
  const allPlayers = new Set(game.players.map((p) => p.id));
  for (const memberId of memberIds) {
    if (!allPlayers.has(memberId)) {
      throw new Error(`玩家 ${memberId} 不在游戏中`);
    }
  }

  // 验证是否有重复队员
  if (new Set(memberIds).size !== memberIds.length) {
    throw new Error("队伍中不能包含重复队员");
  }

  return true;
}
```

### 组队界面组件

```typescript
class TeamSelectionView extends PIXI.Container {
  private game: Game;
  private playerSprites: Map<string, PIXI.Container> = new Map();
  private selectedMembers: Set<string> = new Set();
  private confirmButton: PIXI.Container;

  constructor(game: Game) {
    super();
    this.game = game;

    // 初始化界面
    this.initView();
  }

  private initView(): void {
    // 创建标题
    const title = new PIXI.Text("选择队员", {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
    });
    title.x = 10;
    title.y = 10;
    this.addChild(title);

    // 创建玩家选择区域
    this.createPlayerSelectionArea();

    // 创建确认按钮
    this.confirmButton = this.createConfirmButton();
    this.addChild(this.confirmButton);

    // 更新按钮状态
    this.updateConfirmButtonState();
  }

  private createPlayerSelectionArea(): void {
    // 创建玩家选择区域
    const playerArea = new PIXI.Container();
    playerArea.x = 10;
    playerArea.y = 50;
    this.addChild(playerArea);

    // 添加所有玩家
    this.game.players.forEach((player, index) => {
      const playerSprite = this.createPlayerSprite(player);
      playerSprite.x = (index % 4) * 120;
      playerSprite.y = Math.floor(index / 4) * 150;
      playerArea.addChild(playerSprite);
      this.playerSprites.set(player.id, playerSprite);
    });
  }

  private createPlayerSprite(player: Player): PIXI.Container {
    // 创建玩家精灵
    const container = new PIXI.Container();

    // 添加玩家头像
    const avatar = new PIXI.Sprite(PIXI.Texture.from(player.avatar));
    avatar.width = 80;
    avatar.height = 80;
    container.addChild(avatar);

    // 添加玩家名称
    const name = new PIXI.Text(player.name, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    name.x = 40;
    name.y = 90;
    name.anchor.x = 0.5;
    container.addChild(name);

    // 添加队长标识（如果是当前队长）
    if (player.id === this.game.currentMission.currentCaptainId) {
      const crown = new PIXI.Sprite(PIXI.Texture.from("crown_icon"));
      crown.width = 30;
      crown.height = 30;
      crown.x = 60;
      crown.y = -10;
      container.addChild(crown);
    }

    // 添加交互
    container.interactive = true;
    container.buttonMode = true;
    container.on("pointerdown", () => this.togglePlayerSelection(player.id));

    return container;
  }

  private togglePlayerSelection(playerId: string): void {
    if (this.selectedMembers.has(playerId)) {
      // 取消选择
      this.selectedMembers.delete(playerId);
    } else {
      // 检查是否已达到最大队员数量
      const missionConfig = getMissionConfig(
        this.game.currentMission.missionNumber,
        this.game.players.length
      );

      if (this.selectedMembers.size < missionConfig.requiredTeamSize) {
        // 添加选择
        this.selectedMembers.add(playerId);
      } else {
        // 提示已达到最大队员数量
        this.showMessage("已达到最大队员数量");
        return;
      }
    }

    // 更新玩家选择状态
    this.updatePlayerSelectionState();

    // 更新确认按钮状态
    this.updateConfirmButtonState();
  }

  private updatePlayerSelectionState(): void {
    // 更新所有玩家的选择状态
    this.playerSprites.forEach((sprite, playerId) => {
      const isSelected = this.selectedMembers.has(playerId);

      // 更新选择状态视觉效果
      const avatar = sprite.getChildAt(0) as PIXI.Sprite;
      avatar.tint = isSelected ? 0xffff00 : 0xffffff;

      // 添加或移除选中标记
      if (isSelected && sprite.children.length < 3) {
        const checkmark = new PIXI.Sprite(PIXI.Texture.from("checkmark_icon"));
        checkmark.width = 30;
        checkmark.height = 30;
        checkmark.x = 0;
        checkmark.y = 0;
        sprite.addChild(checkmark);
      } else if (!isSelected && sprite.children.length > 2) {
        sprite.removeChildAt(2);
      }
    });
  }

  private createConfirmButton(): PIXI.Container {
    // 创建确认按钮
    const container = new PIXI.Container();

    // 添加按钮背景
    const background = new PIXI.Graphics();
    background.beginFill(0x3498db);
    background.drawRoundedRect(0, 0, 200, 50, 10);
    background.endFill();
    container.addChild(background);

    // 添加按钮文本
    const text = new PIXI.Text("确认队伍", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    text.x = 100;
    text.y = 25;
    text.anchor.set(0.5);
    container.addChild(text);

    // 添加交互
    container.interactive = true;
    container.buttonMode = true;
    container.on("pointerdown", () => this.confirmTeam());

    // 设置位置
    container.x = 200;
    container.y = 400;

    return container;
  }

  private updateConfirmButtonState(): void {
    // 获取当前任务配置
    const missionConfig = getMissionConfig(
      this.game.currentMission.missionNumber,
      this.game.players.length
    );

    // 检查是否已选择足够的队员
    const isValid =
      this.selectedMembers.size === missionConfig.requiredTeamSize;

    // 更新按钮状态
    this.confirmButton.alpha = isValid ? 1.0 : 0.5;
    this.confirmButton.interactive = isValid;
    this.confirmButton.buttonMode = isValid;
  }

  private confirmTeam(): void {
    // 确认队伍
    try {
      // 验证队伍
      validateTeam(this.game, Array.from(this.selectedMembers));

      // 创建队伍
      const team: Team = {
        captainId: this.game.currentMission.currentCaptainId,
        memberIds: Array.from(this.selectedMembers),
        roundNumber: this.game.currentMission.currentRound,
        missionNumber: this.game.currentMission.missionNumber,
        isApproved: false,
        votes: new Map(),
      };

      // 设置当前队伍
      this.game.currentMission.currentTeam = team;

      // 进入投票阶段
      this.game.currentMission.status = MissionStatus.VOTING;

      // 触发队伍确认事件
      this.emit("teamConfirmed", team);
    } catch (error) {
      // 显示错误信息
      this.showMessage(error.message);
    }
  }

  private showMessage(message: string): void {
    // 显示消息
    console.log(message);
    // 实际实现中应该显示一个消息弹窗
  }
}
```

## 界面原型

```
+------------------------------------------+
|        第2轮任务 - 组队阶段 (队长视角)    |
+------------------------------------------+
|                                          |
|  你是当前队长，请选择3名队员执行任务      |
|  已选择: 1/3                             |
|                                          |
|  所有玩家:                               |
|  +------+  +------+  +------+  +------+  |
|  |玩家1  |  |玩家2  |  |玩家3  |  |玩家4  |  |
|  |[你]   |  |      |  |[已选] |  |      |  |
|  |[队长] |  |      |  |      |  |      |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  +------+  +------+  +------+  +------+  |
|  |玩家5  |  |玩家6  |  |玩家7  |  |玩家8  |  |
|  |      |  |      |  |      |  |      |  |
|  |      |  |      |  |      |  |      |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  已选队员:                               |
|  +------+  +------+                     |
|  |玩家1  |  |玩家3  |                     |
|  |[你]   |  |      |                     |
|  +------+  +------+                     |
|                                          |
|  [查看历史]                 [确认队伍]    |
+------------------------------------------+
```

```
+------------------------------------------+
|      第2轮任务 - 组队阶段 (玩家视角)      |
+------------------------------------------+
|                                          |
|  玩家1(队长)正在选择队员                  |
|  需要选择: 3名队员                        |
|                                          |
|  所有玩家:                               |
|  +------+  +------+  +------+  +------+  |
|  |玩家1  |  |玩家2  |  |玩家3  |  |玩家4  |  |
|  |[队长] |  |      |  |[已选] |  |      |  |
|  |[已选] |  |      |  |      |  |      |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  +------+  +------+  +------+  +------+  |
|  |玩家5  |  |玩家6  |  |玩家7  |  |玩家8  |  |
|  |      |  |      |  |      |  |[你]   |  |
|  |      |  |      |  |      |  |      |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  已选队员:                               |
|  +------+  +------+                     |
|  |玩家1  |  |玩家3  |                     |
|  |[队长] |  |      |                     |
|  +------+  +------+                     |
|                                          |
|  [查看历史]                              |
+------------------------------------------+
```

```
+------------------------------------------+
|             历史队伍记录                  |
+------------------------------------------+
|                                          |
|  任务1 (成功):                           |
|  队长: 玩家8                             |
|  队员: 玩家2, 玩家5, 玩家8               |
|  投票结果: 6赞成 / 2反对                  |
|                                          |
|  任务2 - 第1次尝试 (拒绝):                |
|  队长: 玩家1                             |
|  队员: 玩家1, 玩家3, 玩家6               |
|  投票结果: 3赞成 / 5反对                  |
|                                          |
|  [返回]                                  |
+------------------------------------------+
```

```
+------------------------------------------+
|             队伍确认                      |
+------------------------------------------+
|                                          |
|  你确定要派遣以下队员执行任务吗？          |
|                                          |
|  +------+  +------+  +------+           |
|  |玩家1  |  |玩家3  |  |玩家6  |           |
|  |[你]   |  |      |  |      |           |
|  |[队长] |  |      |  |      |           |
|  +------+  +------+  +------+           |
|                                          |
|  确认后，所有玩家将对此队伍进行投票        |
|                                          |
|  [取消]                    [确认]         |
+------------------------------------------+
```

## 验收标准

1. 队长轮换机制正确，按照玩家顺序轮换
2. 队长能够选择正确数量的队员
3. 队伍验证逻辑能够正确识别无效队伍
4. 界面清晰展示当前队长和已选队员
5. 队伍历史记录功能正常工作
6. 断线重连后能够恢复当前组队状态
7. 界面响应迅速，无明显延迟

## 技术依赖

- PIXI.js 渲染引擎
- 游戏状态管理系统
- 网络同步模块
- 用户界面组件库

## 工作量估计

1 人天

## 相关文档

- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
- [任务系统技术方案](./技术方案.md)
