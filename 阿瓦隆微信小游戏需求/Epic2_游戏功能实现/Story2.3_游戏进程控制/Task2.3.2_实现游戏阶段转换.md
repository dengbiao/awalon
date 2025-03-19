# Task 2.3.2: 实现游戏阶段转换

## 描述

实现阿瓦隆游戏的阶段转换系统，负责管理游戏从一个阶段到另一个阶段的平滑过渡，包括阶段转换逻辑、转换条件验证、转换事件触发和界面切换效果。

## 详细需求

### 功能需求

1. 阶段转换管理：

   - 定义游戏的所有阶段（等待开始、角色分配、角色揭示、任务阶段、刺杀阶段、结果展示）
   - 实现阶段之间的转换规则和条件
   - 提供阶段转换的触发接口
   - 处理阶段转换的异常情况和回退机制

2. 阶段转换事件：

   - 在阶段转换前触发预转换事件，允许其他模块进行准备
   - 在阶段转换后触发后转换事件，通知其他模块进行相应处理
   - 支持阶段特定的初始化和清理事件
   - 提供阶段转换的历史记录

3. 界面和交互控制：
   - 根据当前阶段显示相应的界面元素
   - 管理不同阶段的交互权限和可用操作
   - 实现阶段转换的动画和过渡效果
   - 提供阶段状态的指示和提示

### 技术要求

1. 实现可扩展的阶段转换器，支持新阶段的添加
2. 设计合理的状态机结构，表示阶段和转换关系
3. 确保阶段转换的原子性和一致性
4. 优化阶段转换的性能，减少转换延迟
5. 实现阶段转换的日志记录和错误追踪
6. 支持断线重连时的阶段恢复和同步

## 实现步骤

1. 设计阶段状态机和转换规则
2. 实现阶段控制器核心类
3. 实现阶段转换事件系统
4. 实现阶段特定的初始化逻辑
5. 实现界面和交互控制
6. 实现阶段转换动画和效果
7. 进行单元测试和集成测试

## 代码示例

### 阶段状态机定义

```typescript
// 游戏阶段枚举
enum GamePhase {
  WAITING = "waiting",
  ROLE_ASSIGNMENT = "role_assignment",
  ROLE_REVEAL = "role_reveal",
  MISSION = "mission",
  ASSASSINATION = "assassination",
  RESULT = "result",
  ENDED = "ended",
}

// 阶段转换规则
interface PhaseTransition {
  from: GamePhase;
  to: GamePhase;
  condition: (gameState: GameState) => boolean;
  action: (gameState: GameState) => Promise<void>;
}

// 阶段转换事件
interface PhaseTransitionEvent {
  previousPhase: GamePhase;
  currentPhase: GamePhase;
  reason: string;
  timestamp: Date;
}
```

### 阶段控制器实现

```typescript
class GamePhaseController {
  private gameStateManager: GameStateManager;
  private transitionRules: PhaseTransition[];
  private phaseInitializers: Map<
    GamePhase,
    (gameState: GameState) => Promise<void>
  >;
  private eventEmitter: EventEmitter;

  constructor(gameStateManager: GameStateManager) {
    this.gameStateManager = gameStateManager;
    this.eventEmitter = new EventEmitter();
    this.transitionRules = this.defineTransitionRules();
    this.phaseInitializers = this.definePhaseInitializers();

    // 订阅游戏状态更新事件
    this.gameStateManager.subscribeToStateChanges(
      this.handleStateChange.bind(this)
    );
  }

  // 定义阶段转换规则
  private defineTransitionRules(): PhaseTransition[] {
    return [
      // 等待开始 -> 角色分配
      {
        from: GamePhase.WAITING,
        to: GamePhase.ROLE_ASSIGNMENT,
        condition: (state) => state.players.every((p) => p.isReady),
        action: async (state) => {
          console.log("Starting role assignment");
        },
      },
      // 角色分配 -> 角色揭示
      {
        from: GamePhase.ROLE_ASSIGNMENT,
        to: GamePhase.ROLE_REVEAL,
        condition: (state) => state.playerRoles.size === state.players.length,
        action: async (state) => {
          console.log("Starting role reveal");
        },
      },
      // 角色揭示 -> 任务阶段
      {
        from: GamePhase.ROLE_REVEAL,
        to: GamePhase.MISSION,
        condition: (state) => state.players.every((p) => p.roleConfirmed),
        action: async (state) => {
          console.log("Starting mission phase");
        },
      },
      // 任务阶段 -> 刺杀阶段
      {
        from: GamePhase.MISSION,
        to: GamePhase.ASSASSINATION,
        condition: (state) => {
          const successCount = state.missionResults.filter(
            (r) => r.result === "SUCCESS"
          ).length;
          return successCount >= 3;
        },
        action: async (state) => {
          console.log("Starting assassination phase");
        },
      },
      // 任务阶段 -> 结果阶段 (坏人获胜)
      {
        from: GamePhase.MISSION,
        to: GamePhase.RESULT,
        condition: (state) => {
          const failCount = state.missionResults.filter(
            (r) => r.result === "FAIL"
          ).length;
          return failCount >= 3 || state.consecutiveRejections >= 5;
        },
        action: async (state) => {
          console.log("Evil wins, showing results");
        },
      },
      // 刺杀阶段 -> 结果阶段
      {
        from: GamePhase.ASSASSINATION,
        to: GamePhase.RESULT,
        condition: (state) => state.assassinationTarget !== null,
        action: async (state) => {
          console.log("Assassination complete, showing results");
        },
      },
      // 结果阶段 -> 结束
      {
        from: GamePhase.RESULT,
        to: GamePhase.ENDED,
        condition: (state) => true, // 可以随时结束
        action: async (state) => {
          console.log("Game ended");
        },
      },
    ];
  }

  // 定义阶段初始化函数
  private definePhaseInitializers(): Map<
    GamePhase,
    (gameState: GameState) => Promise<void>
  > {
    const initializers = new Map();

    // 角色分配阶段初始化
    initializers.set(GamePhase.ROLE_ASSIGNMENT, async (gameState) => {
      // 初始化角色分配逻辑
    });

    // 任务阶段初始化
    initializers.set(GamePhase.MISSION, async (gameState) => {
      // 初始化第一个任务
      if (gameState.currentMissionNumber === 0) {
        // 更新为第一个任务
        this.gameStateManager.updateGameState({
          currentMissionNumber: 1,
        });
      }
    });

    // 其他阶段初始化...

    return initializers;
  }

  // 尝试转换到指定阶段
  public async transitionTo(
    targetPhase: GamePhase,
    reason: string
  ): Promise<boolean> {
    const gameState = this.gameStateManager.getGameState();
    const currentPhase = gameState.status;

    // 寻找匹配的转换规则
    const rule = this.transitionRules.find(
      (r) =>
        r.from === currentPhase &&
        r.to === targetPhase &&
        r.condition(gameState)
    );

    if (!rule) {
      console.error(
        `Invalid phase transition: ${currentPhase} -> ${targetPhase}`
      );
      return false;
    }

    try {
      // 触发预转换事件
      this.eventEmitter.emit("beforePhaseTransition", {
        from: currentPhase,
        to: targetPhase,
        reason,
      });

      // 执行转换动作
      await rule.action(gameState);

      // 更新游戏状态
      this.gameStateManager.updateGameState({
        status: targetPhase,
      });

      // 执行新阶段初始化
      const initializer = this.phaseInitializers.get(targetPhase);
      if (initializer) {
        await initializer(this.gameStateManager.getGameState());
      }

      // 触发后转换事件
      const event: PhaseTransitionEvent = {
        previousPhase: currentPhase,
        currentPhase: targetPhase,
        reason,
        timestamp: new Date(),
      };

      this.eventEmitter.emit("afterPhaseTransition", event);

      return true;
    } catch (error) {
      console.error(`Error during phase transition: ${error}`);
      return false;
    }
  }

  // 处理游戏状态变化
  private handleStateChange(update: {
    previousState: GameState;
    currentState: GameState;
  }): void {
    // 检查是否有自动转换的条件满足
    this.checkAutomaticTransitions(update.currentState);
  }

  // 检查自动转换条件
  private async checkAutomaticTransitions(gameState: GameState): Promise<void> {
    const currentPhase = gameState.status;

    // 寻找满足条件的自动转换
    for (const rule of this.transitionRules) {
      if (rule.from === currentPhase && rule.condition(gameState)) {
        // 自动执行转换
        await this.transitionTo(rule.to, "自动转换");
        break;
      }
    }
  }

  // 订阅阶段转换事件
  public onPhaseTransition(
    callback: (event: PhaseTransitionEvent) => void
  ): () => void {
    this.eventEmitter.on("afterPhaseTransition", callback);
    return () => this.eventEmitter.off("afterPhaseTransition", callback);
  }
}
```

### 阶段界面管理器

```typescript
class PhaseUIManager {
  private gameStateManager: GameStateManager;
  private phaseController: GamePhaseController;
  private phaseViews: Map<GamePhase, PIXI.Container>;
  private currentView: PIXI.Container | null = null;
  private app: PIXI.Application;

  constructor(
    app: PIXI.Application,
    gameStateManager: GameStateManager,
    phaseController: GamePhaseController
  ) {
    this.app = app;
    this.gameStateManager = gameStateManager;
    this.phaseController = phaseController;
    this.phaseViews = this.createPhaseViews();

    // 订阅阶段转换事件
    this.phaseController.onPhaseTransition(
      this.handlePhaseTransition.bind(this)
    );
  }

  // 创建各阶段的视图
  private createPhaseViews(): Map<GamePhase, PIXI.Container> {
    const views = new Map();

    // 等待阶段视图
    views.set(GamePhase.WAITING, this.createWaitingView());

    // 角色分配阶段视图
    views.set(GamePhase.ROLE_ASSIGNMENT, this.createRoleAssignmentView());

    // 角色揭示阶段视图
    views.set(GamePhase.ROLE_REVEAL, this.createRoleRevealView());

    // 任务阶段视图
    views.set(GamePhase.MISSION, this.createMissionView());

    // 刺杀阶段视图
    views.set(GamePhase.ASSASSINATION, this.createAssassinationView());

    // 结果阶段视图
    views.set(GamePhase.RESULT, this.createResultView());

    return views;
  }

  // 处理阶段转换
  private handlePhaseTransition(event: PhaseTransitionEvent): void {
    // 获取新阶段的视图
    const newView = this.phaseViews.get(event.currentPhase);

    if (!newView) {
      console.error(`No view found for phase: ${event.currentPhase}`);
      return;
    }

    // 执行视图切换动画
    this.transitionViews(this.currentView, newView, event);
  }

  // 视图切换动画
  private transitionViews(
    oldView: PIXI.Container | null,
    newView: PIXI.Container,
    event: PhaseTransitionEvent
  ): void {
    // 如果当前有视图，先淡出
    if (oldView && oldView.parent) {
      gsap.to(oldView, {
        alpha: 0,
        duration: 0.5,
        onComplete: () => {
          oldView.parent?.removeChild(oldView);
          this.showNewView(newView);
        },
      });
    } else {
      this.showNewView(newView);
    }

    // 展示阶段转换提示
    this.showPhaseTransitionHint(event);
  }

  // 显示新视图
  private showNewView(view: PIXI.Container): void {
    // 重置新视图的状态
    view.alpha = 0;
    this.app.stage.addChild(view);

    // 淡入新视图
    gsap.to(view, {
      alpha: 1,
      duration: 0.5,
    });

    this.currentView = view;
  }

  // 显示阶段转换提示
  private showPhaseTransitionHint(event: PhaseTransitionEvent): void {
    const hint = new PIXI.Container();

    // 创建背景
    const bg = new PIXI.Graphics();
    bg.beginFill(0x000000, 0.7);
    bg.drawRect(0, 0, 500, 100);
    bg.endFill();
    hint.addChild(bg);

    // 创建文本
    const text = new PIXI.Text(
      `${this.getPhaseDisplayName(event.currentPhase)}`,
      {
        fontFamily: "Arial",
        fontSize: 24,
        fill: 0xffffff,
      }
    );
    text.x = 250;
    text.y = 40;
    text.anchor.set(0.5);
    hint.addChild(text);

    // 定位提示
    hint.x = this.app.screen.width / 2 - 250;
    hint.y = this.app.screen.height / 2 - 50;
    hint.alpha = 0;

    this.app.stage.addChild(hint);

    // 动画显示然后隐藏
    gsap
      .timeline()
      .to(hint, { alpha: 1, duration: 0.3 })
      .to(hint, {
        alpha: 0,
        duration: 0.3,
        delay: 1.5,
        onComplete: () => {
          this.app.stage.removeChild(hint);
        },
      });
  }

  // 获取阶段显示名称
  private getPhaseDisplayName(phase: GamePhase): string {
    const phaseNames = {
      [GamePhase.WAITING]: "等待开始",
      [GamePhase.ROLE_ASSIGNMENT]: "角色分配",
      [GamePhase.ROLE_REVEAL]: "角色揭示",
      [GamePhase.MISSION]: "任务阶段",
      [GamePhase.ASSASSINATION]: "刺杀阶段",
      [GamePhase.RESULT]: "结果展示",
      [GamePhase.ENDED]: "游戏结束",
    };

    return phaseNames[phase] || "未知阶段";
  }

  // 简单示例：创建等待阶段视图
  private createWaitingView(): PIXI.Container {
    const container = new PIXI.Container();

    // 实现等待阶段的界面
    // ...

    return container;
  }

  // 其他阶段视图创建方法
  private createRoleAssignmentView(): PIXI.Container {
    // 实现
    return new PIXI.Container();
  }

  private createRoleRevealView(): PIXI.Container {
    // 实现
    return new PIXI.Container();
  }

  private createMissionView(): PIXI.Container {
    // 实现
    return new PIXI.Container();
  }

  private createAssassinationView(): PIXI.Container {
    // 实现
    return new PIXI.Container();
  }

  private createResultView(): PIXI.Container {
    // 实现
    return new PIXI.Container();
  }
}
```

## 界面原型

```
+------------------------------------------+
|           阶段转换提示                    |
+------------------------------------------+
|                                          |
|  正在进入: 角色分配阶段                   |
|                                          |
|  [进度指示器]                            |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           阶段状态指示器                  |
+------------------------------------------+
|                                          |
|  [准备] -> [角色] -> [任务] -> [刺杀] -> [结果]
|                     ^                    |
|                  当前阶段                 |
|                                          |
+------------------------------------------+
```

```
+------------------------------------------+
|           阶段转换失败提示                |
+------------------------------------------+
|                                          |
|  无法进入任务阶段                         |
|                                          |
|  原因: 有玩家尚未确认角色                  |
|                                          |
|  请等待所有玩家确认后继续                  |
|                                          |
|            [确认]                        |
+------------------------------------------+
```

## 验收标准

1. 所有游戏阶段之间的转换符合游戏规则
2. 阶段转换条件验证准确无误
3. 转换事件正确触发，相关模块能够响应
4. 界面根据阶段变化正确切换
5. 阶段转换动画流畅自然
6. 断线重连后能够恢复到正确的阶段
7. 非法的阶段转换被阻止并给出明确提示
8. 阶段转换日志记录完整，便于追踪问题

## 技术依赖

- TypeScript/JavaScript
- PIXI.js 渲染引擎
- GSAP 动画库
- 游戏状态管理系统
- 事件发布-订阅系统

## 工作量估计

1 人天

## 相关文档

- [游戏进程控制技术方案](./技术方案.md)
- [游戏状态管理](./Task2.3.1_实现游戏状态管理.md)
- [阿瓦隆游戏规则](../../阿瓦隆游戏规则.md)
