# Task 2.2.2: 实现投票系统

## 描述

实现阿瓦隆游戏中的投票系统，允许玩家对队长提出的队伍进行投票，并根据投票结果决定是否批准队伍执行任务。

## 详细需求

### 功能需求

1. 投票机制：

   - 队长确认队伍后，所有玩家进行投票
   - 每位玩家可以选择赞成或拒绝
   - 投票结果由多数决定（赞成票多于拒绝票则通过）
   - 投票结果公开展示给所有玩家
   - 支持投票倒计时功能

2. 连续拒绝机制：

   - 记录连续拒绝的次数
   - 当连续拒绝达到 5 次时，坏人自动获胜
   - 提供连续拒绝警告提示

3. 投票结果处理：
   - 投票通过后，进入任务执行阶段
   - 投票拒绝后，轮换到下一位队长重新组队
   - 记录每次投票的详细信息，包括每位玩家的投票选择

### 界面要求

1. 投票卡片设计：

   - 提供明显区分的赞成/拒绝投票卡片
   - 卡片设计美观，符合游戏风格
   - 选择后有明确的视觉反馈

2. 投票状态展示：

   - 显示已投票和未投票的玩家
   - 显示投票倒计时
   - 投票结束前不显示其他玩家的选择

3. 投票结果展示：

   - 清晰展示投票结果（通过/拒绝）
   - 显示赞成和拒绝的票数
   - 显示每位玩家的投票选择
   - 提供适当的动画效果

4. 连续拒绝提示：
   - 显示当前连续拒绝次数
   - 接近 5 次时提供警告提示
   - 达到 5 次时显示游戏结束提示

### 技术要求

1. 实现安全的投票收集机制，防止作弊
2. 确保所有玩家都能看到相同的投票结果
3. 处理玩家断线或超时情况
4. 优化网络传输，减少延迟
5. 实现投票历史记录功能

## 实现步骤

1. 设计投票数据结构和状态管理
2. 实现投票界面组件
3. 实现投票收集和统计逻辑
4. 实现投票结果展示
5. 实现连续拒绝计数和处理
6. 实现投票历史记录
7. 进行测试和优化

## 代码示例

### 投票收集逻辑

```typescript
// 玩家投票
function playerVote(game: Game, playerId: string, vote: VoteType): void {
  // 验证游戏状态
  if (game.currentMission.status !== MissionStatus.VOTING) {
    throw new Error("当前不是投票阶段");
  }

  // 验证玩家是否已投票
  if (game.currentMission.currentTeam.votes.has(playerId)) {
    throw new Error("玩家已经投过票");
  }

  // 记录玩家投票
  game.currentMission.currentTeam.votes.set(playerId, vote);

  // 检查是否所有玩家都已投票
  if (game.currentMission.currentTeam.votes.size === game.players.length) {
    // 计算投票结果
    calculateVoteResult(game);
  }
}
```

### 投票结果计算

```typescript
// 计算投票结果
function calculateVoteResult(game: Game): void {
  const votes = game.currentMission.currentTeam.votes;
  let approveCount = 0;
  let rejectCount = 0;

  // 统计赞成和拒绝票数
  votes.forEach((vote) => {
    if (vote === VoteType.APPROVE) approveCount++;
    else rejectCount++;
  });

  // 判断投票结果
  const isApproved = approveCount > rejectCount;
  game.currentMission.currentTeam.isApproved = isApproved;

  if (isApproved) {
    // 队伍获得批准，进入任务执行阶段
    game.currentMission.status = MissionStatus.EXECUTING;
    game.consecutiveRejections = 0; // 重置连续拒绝计数
  } else {
    // 队伍被拒绝，更新连续拒绝计数
    game.consecutiveRejections++;

    // 检查是否达到连续拒绝上限
    if (game.consecutiveRejections >= 5) {
      // 坏人获胜
      endGameWithEvilVictory(game);
    } else {
      // 记录被拒绝的队伍
      game.currentMission.rejectedTeams.push(game.currentMission.currentTeam);

      // 轮换到下一位队长
      game.currentMission.currentRound++;
      selectNextCaptain(game);

      // 重新进入组队阶段
      game.currentMission.status = MissionStatus.TEAM_BUILDING;
      game.currentMission.currentTeam = null;
    }
  }

  // 触发投票结果事件
  emitVoteResultEvent(game);
}
```

### 投票界面组件

```typescript
class VotingView extends PIXI.Container {
  private game: Game;
  private approveCard: PIXI.Container;
  private rejectCard: PIXI.Container;
  private selectedVote: VoteType | null = null;
  private countdownText: PIXI.Text;
  private countdownTimer: number;

  constructor(game: Game) {
    super();
    this.game = game;

    // 初始化界面
    this.initView();

    // 开始倒计时
    this.startCountdown(30); // 30秒倒计时
  }

  private initView(): void {
    // 创建标题
    const title = new PIXI.Text("投票阶段", {
      fontFamily: "Arial",
      fontSize: 24,
      fill: 0xffffff,
    });
    title.x = 10;
    title.y = 10;
    this.addChild(title);

    // 创建队伍信息
    this.createTeamInfo();

    // 创建投票卡片
    this.approveCard = this.createVoteCard(VoteType.APPROVE);
    this.approveCard.x = 100;
    this.approveCard.y = 200;
    this.addChild(this.approveCard);

    this.rejectCard = this.createVoteCard(VoteType.REJECT);
    this.rejectCard.x = 300;
    this.rejectCard.y = 200;
    this.addChild(this.rejectCard);

    // 创建倒计时文本
    this.countdownText = new PIXI.Text("30", {
      fontFamily: "Arial",
      fontSize: 20,
      fill: 0xffffff,
    });
    this.countdownText.x = 200;
    this.countdownText.y = 350;
    this.addChild(this.countdownText);

    // 创建投票状态显示
    this.createVotingStatus();
  }

  private createTeamInfo(): void {
    // 创建队伍信息容器
    const container = new PIXI.Container();
    container.x = 10;
    container.y = 50;
    this.addChild(container);

    // 添加队长信息
    const captainId = this.game.currentMission.currentCaptainId;
    const captain = this.game.players.find((p) => p.id === captainId);

    const captainText = new PIXI.Text(`队长: ${captain.name}`, {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    container.addChild(captainText);

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

    // 添加任务信息
    const missionText = new PIXI.Text(
      `任务 ${this.game.currentMission.missionNumber} - 轮次 ${this.game.currentMission.currentRound}`,
      {
        fontFamily: "Arial",
        fontSize: 16,
        fill: 0xffffff,
      }
    );
    missionText.y = 50;
    container.addChild(missionText);

    // 添加连续拒绝信息
    if (this.game.consecutiveRejections > 0) {
      const rejectText = new PIXI.Text(
        `连续拒绝: ${this.game.consecutiveRejections}/5`,
        {
          fontFamily: "Arial",
          fontSize: 16,
          fill: this.game.consecutiveRejections >= 3 ? 0xff0000 : 0xffffff,
        }
      );
      rejectText.y = 75;
      container.addChild(rejectText);
    }
  }

  private createVoteCard(voteType: VoteType): PIXI.Container {
    // 创建投票卡片容器
    const container = new PIXI.Container();

    // 添加卡片背景
    const background = new PIXI.Graphics();
    background.beginFill(voteType === VoteType.APPROVE ? 0x2ecc71 : 0xe74c3c);
    background.drawRoundedRect(0, 0, 150, 100, 10);
    background.endFill();
    container.addChild(background);

    // 添加卡片图标
    const icon = new PIXI.Sprite(
      PIXI.Texture.from(
        voteType === VoteType.APPROVE ? "approve_icon" : "reject_icon"
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
      voteType === VoteType.APPROVE ? "赞成" : "拒绝",
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
    container.on("pointerdown", () => this.selectVote(voteType));

    return container;
  }

  private createVotingStatus(): void {
    // 创建投票状态容器
    const container = new PIXI.Container();
    container.x = 10;
    container.y = 400;
    this.addChild(container);

    // 添加标题
    const title = new PIXI.Text("投票状态:", {
      fontFamily: "Arial",
      fontSize: 16,
      fill: 0xffffff,
    });
    container.addChild(title);

    // 添加玩家投票状态
    this.game.players.forEach((player, index) => {
      const hasVoted = this.game.currentMission.currentTeam.votes.has(
        player.id
      );

      const statusText = new PIXI.Text(
        `${player.name}: ${hasVoted ? "已投票" : "未投票"}`,
        {
          fontFamily: "Arial",
          fontSize: 14,
          fill: hasVoted ? 0x2ecc71 : 0xe74c3c,
        }
      );
      statusText.x = (index % 2) * 200;
      statusText.y = 25 + Math.floor(index / 2) * 20;
      container.addChild(statusText);
    });
  }

  private selectVote(voteType: VoteType): void {
    // 如果已经投票，则不允许再次选择
    if (this.selectedVote !== null) {
      return;
    }

    // 记录选择
    this.selectedVote = voteType;

    // 更新卡片视觉效果
    this.approveCard.alpha = voteType === VoteType.APPROVE ? 1.0 : 0.5;
    this.rejectCard.alpha = voteType === VoteType.REJECT ? 1.0 : 0.5;

    // 添加选中效果
    const selectedCard =
      voteType === VoteType.APPROVE ? this.approveCard : this.rejectCard;
    const glow = new PIXI.filters.GlowFilter();
    glow.color = 0xffffff;
    glow.distance = 15;
    glow.outerStrength = 2;
    selectedCard.filters = [glow];

    // 发送投票
    try {
      playerVote(this.game, this.game.currentPlayerId, voteType);

      // 更新投票状态显示
      this.updateVotingStatus();
    } catch (error) {
      // 显示错误信息
      this.showMessage(error.message);
    }
  }

  private updateVotingStatus(): void {
    // 移除旧的投票状态显示
    this.removeChild(this.getChildAt(this.children.length - 1));

    // 创建新的投票状态显示
    this.createVotingStatus();
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

        // 如果还没投票，自动投票
        if (this.selectedVote === null) {
          this.handleTimeoutVote();
        }
      } else {
        // 更新倒计时
        this.countdownText.text = (currentTime - 1).toString();
      }
    }, 1000);
  }

  private handleTimeoutVote(): void {
    // 超时自动投票（可以根据游戏规则设置默认投票）
    const defaultVote = VoteType.REJECT; // 默认拒绝
    this.selectVote(defaultVote);
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
|        第2轮任务 - 投票阶段              |
+------------------------------------------+
|                                          |
|  队长: 玩家1                             |
|  队员: 玩家1, 玩家3, 玩家6               |
|  任务2 - 轮次1                           |
|  连续拒绝: 0/5                           |
|                                          |
|  请投票是否批准该队伍执行任务:            |
|                                          |
|  +---------------+  +---------------+    |
|  |               |  |               |    |
|  |     [图标]     |  |     [图标]     |    |
|  |               |  |               |    |
|  |     赞成      |  |     拒绝      |    |
|  |               |  |               |    |
|  +---------------+  +---------------+    |
|                                          |
|  倒计时: 23                              |
|                                          |
|  投票状态:                               |
|  玩家1(你): 已投票    玩家2: 未投票      |
|  玩家3: 已投票       玩家4: 已投票      |
|  玩家5: 未投票       玩家6: 已投票      |
|  玩家7: 未投票       玩家8: 已投票      |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           投票结果 - 队伍获得批准         |
+------------------------------------------+
|                                          |
|  队长: 玩家1                             |
|  队员: 玩家1, 玩家3, 玩家6               |
|  任务2 - 轮次1                           |
|                                          |
|  投票结果: 批准 (5票赞成 / 3票拒绝)       |
|                                          |
|  玩家投票详情:                           |
|  +------+  +------+  +------+  +------+  |
|  |玩家1  |  |玩家2  |  |玩家3  |  |玩家4  |  |
|  |[赞成] |  |[拒绝] |  |[赞成] |  |[赞成] |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  +------+  +------+  +------+  +------+  |
|  |玩家5  |  |玩家6  |  |玩家7  |  |玩家8  |  |
|  |[拒绝] |  |[赞成] |  |[拒绝] |  |[赞成] |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  队伍获得批准，将进入任务执行阶段         |
|                                          |
|                [继续]                    |
+------------------------------------------+
```

```
+------------------------------------------+
|           投票结果 - 队伍被拒绝           |
+------------------------------------------+
|                                          |
|  队长: 玩家1                             |
|  队员: 玩家1, 玩家3, 玩家6               |
|  任务2 - 轮次1                           |
|                                          |
|  投票结果: 拒绝 (3票赞成 / 5票拒绝)       |
|                                          |
|  玩家投票详情:                           |
|  +------+  +------+  +------+  +------+  |
|  |玩家1  |  |玩家2  |  |玩家3  |  |玩家4  |  |
|  |[赞成] |  |[拒绝] |  |[赞成] |  |[拒绝] |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  +------+  +------+  +------+  +------+  |
|  |玩家5  |  |玩家6  |  |玩家7  |  |玩家8  |  |
|  |[拒绝] |  |[赞成] |  |[拒绝] |  |[拒绝] |  |
|  +------+  +------+  +------+  +------+  |
|                                          |
|  队伍被拒绝，玩家2将成为新队长            |
|  连续拒绝: 1/5                           |
|                                          |
|                [继续]                    |
+------------------------------------------+
```

```
+------------------------------------------+
|             连续拒绝警告                  |
+------------------------------------------+
|                                          |
|  ⚠️ 警告 ⚠️                              |
|                                          |
|  队伍已连续被拒绝4次！                    |
|                                          |
|  如果下一次队伍再次被拒绝，               |
|  游戏将以坏人胜利结束！                   |
|                                          |
|  请谨慎投票！                            |
|                                          |
|                [确认]                    |
+------------------------------------------+
```

## 验收标准

1. 投票界面清晰展示队伍信息和投票选项
2. 所有玩家都能成功进行投票
3. 投票结果正确计算并展示
4. 连续拒绝机制正常工作，达到 5 次时坏人获胜
5. 投票超时处理机制有效
6. 断线重连后能够恢复当前投票状态
7. 投票历史记录功能正常工作

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
