# Task 1.3.5: 实现游戏状态和进度展示

## 任务描述

设计并实现阿瓦隆游戏中的游戏状态和进度展示功能，为玩家提供清晰直观的游戏进程信息，包括当前游戏阶段、任务进度、投票记录和关键事件提示。该功能将贯穿整个游戏过程，帮助玩家理解当前游戏状态并做出相应决策。

## 详细需求

### 功能需求

1. **游戏阶段指示器**

   - 清晰显示当前游戏所处的阶段（如角色展示、队长选择、组队、投票、任务执行等）
   - 提供阶段转换的视觉反馈和动画效果
   - 显示当前阶段的倒计时（如适用）

2. **任务进度板**

   - 展示 5 轮任务的状态（未开始、进行中、成功、失败）
   - 显示当前任务轮次和重要性（所需队员数量）
   - 标记当前任务的队长
   - 展示历史任务的成功/失败记录

3. **投票历史记录**

   - 记录并展示每轮组队投票的结果
   - 显示连续拒绝的次数（接近 5 次拒绝时有警告）
   - 提供查看详细投票历史的功能

4. **玩家状态指示**

   - 显示所有玩家的基本状态（在线/离线、是否为当前队长等）
   - 标记当前任务队员
   - 显示投票和任务执行的完成状态

5. **事件通知系统**

   - 提供重要游戏事件的通知（如新的队长、投票开始、任务结果等）
   - 关键事件有醒目的视觉和音效提示
   - 支持临时性提示和持久性状态显示

6. **游戏统计信息**
   - 显示当前游戏进行的时间
   - 提供成功/失败任务的统计
   - 展示投票统计（通过/拒绝次数）

### 界面要求

1. **整体布局**

   - 设计紧凑而信息丰富的状态区域，不占用过多游戏界面空间
   - 支持折叠/展开详细信息的交互
   - 确保关键状态信息始终可见

2. **任务进度板设计**

   - 使用直观的视觉元素表示任务状态
   - 成功/失败任务有明确的色彩和图标区分
   - 当前任务有醒目的标识

3. **阶段指示器设计**

   - 使用图标和文字结合的方式表示游戏阶段
   - 阶段转换有平滑的动画效果
   - 当前阶段有醒目的高亮效果

4. **通知设计**

   - 重要通知使用醒目但不干扰游戏的方式呈现
   - 支持通知的分级显示（重要/一般/次要）
   - 通知有适当的显示时长和消失动画

5. **响应式设计**
   - 适配不同屏幕尺寸的设备
   - 在小屏幕设备上优先显示最重要的信息
   - 支持横屏和竖屏模式

### 技术要求

1. **状态管理**

   - 实现高效的游戏状态管理机制
   - 确保状态更新的实时性和准确性
   - 处理状态变化的动画和过渡效果

2. **实时更新**

   - 实时同步服务器的游戏状态到所有玩家
   - 确保所有玩家看到一致的游戏进度
   - 处理网络延迟和状态同步问题

3. **性能优化**

   - 优化状态更新的性能，减少不必要的渲染
   - 确保动画效果不影响游戏性能
   - 适配低端设备的性能表现

4. **异常处理**
   - 处理网络异常情况下的状态恢复
   - 提供状态不一致时的修复机制
   - 在关键状态变化失败时提供重试机制

## 实现步骤

1. **设计状态展示组件**

   - 设计任务进度板的视觉样式
   - 设计阶段指示器的交互方式
   - 规划通知系统的显示逻辑

2. **开发基础状态管理模块**

   - 实现游戏状态的数据结构
   - 开发状态更新和订阅机制
   - 实现状态变化的动画控制

3. **开发任务进度板组件**

   - 实现任务状态的可视化展示
   - 开发任务历史记录的查看功能
   - 实现任务状态变化的动画效果

4. **开发阶段指示器组件**

   - 实现游戏阶段的可视化展示
   - 开发阶段转换的动画效果
   - 实现倒计时功能（如适用）

5. **开发通知系统**

   - 实现不同级别通知的显示逻辑
   - 开发通知队列和管理机制
   - 实现通知的动画效果

6. **集成和优化**
   - 将各组件集成到游戏主界面
   - 优化组件间的协作和状态共享
   - 测试不同游戏场景下的状态展示

## 数据结构

```typescript
// 游戏阶段枚举
enum GamePhase {
  ROLE_REVEAL = "roleReveal", // 角色展示
  TEAM_SELECTION = "teamSelection", // 队长选择队员
  TEAM_VOTING = "teamVoting", // 队伍投票
  MISSION_EXECUTION = "missionExecution", // 任务执行
  GAME_OVER = "gameOver", // 游戏结束
}

// 任务状态枚举
enum MissionStatus {
  NOT_STARTED = "notStarted", // 未开始
  IN_PROGRESS = "inProgress", // 进行中
  SUCCESS = "success", // 成功
  FAILURE = "failure", // 失败
}

// 游戏状态接口
interface GameState {
  gameId: string; // 游戏ID
  currentPhase: GamePhase; // 当前游戏阶段
  currentRound: number; // 当前轮次（1-5）
  currentLeader: string; // 当前队长ID
  rejectionCount: number; // 连续拒绝次数
  missions: MissionState[]; // 任务状态数组
  players: PlayerState[]; // 玩家状态数组
  teamMembers: string[]; // 当前任务队员ID数组
  phaseStartTime: number; // 当前阶段开始时间戳
  phaseTimeLimit?: number; // 当前阶段时间限制（秒）
}

// 任务状态接口
interface MissionState {
  missionNumber: number; // 任务轮次（1-5）
  requiredPlayers: number; // 所需队员数量
  status: MissionStatus; // 任务状态
  leader?: string; // 队长ID
  teamMembers?: string[]; // 队员ID数组
  votingResults?: VotingResult; // 投票结果
  missionResults?: MissionResult; // 任务结果
}

// 玩家状态接口
interface PlayerState {
  playerId: string; // 玩家ID
  playerName: string; // 玩家名称
  isOnline: boolean; // 是否在线
  isLeader: boolean; // 是否为当前队长
  isOnTeam: boolean; // 是否为当前队员
  hasVoted: boolean; // 是否已投票
  hasExecuted: boolean; // 是否已执行任务
}

// 投票结果接口
interface VotingResult {
  approved: boolean; // 是否通过
  approveCount: number; // 赞成票数
  rejectCount: number; // 反对票数
  votes: Vote[]; // 详细投票记录
}

// 投票记录接口
interface Vote {
  playerId: string; // 玩家ID
  approved: boolean; // 是否赞成
}

// 任务结果接口
interface MissionResult {
  success: boolean; // 是否成功
  successCount: number; // 成功票数
  failureCount: number; // 失败票数
}

// 通知接口
interface GameNotification {
  id: string; // 通知ID
  type: "info" | "warning" | "error" | "success"; // 通知类型
  message: string; // 通知内容
  duration?: number; // 显示时长（毫秒）
  timestamp: number; // 创建时间戳
  isPersistent: boolean; // 是否为持久通知
}
```

## 界面原型

```
+------------------------------------------+
|                游戏状态区                  |
+------------------------------------------+
| 当前阶段: [队长选择队员] - 剩余时间: 01:45  |
|                                          |
| 任务进度:                                 |
| [成功] [失败] [进行中] [未开始] [未开始]    |
| 任务1   任务2   任务3    任务4    任务5    |
| 2人    3人     3人      4人      4人      |
|                                          |
| 连续拒绝: 2/5                             |
|                                          |
| 玩家状态:                                 |
| [玩家1]* [玩家2] [玩家3]√ [玩家4]√ [玩家5] |
| [玩家6]  [玩家7] [玩家8]                  |
| * = 队长, √ = 已选队员                    |
|                                          |
| 游戏时间: 15:23 | 成功/失败: 1/1           |
+------------------------------------------+
|            最近通知                       |
| > 玩家1成为新的队长                        |
| > 玩家1正在选择队员...                     |
+------------------------------------------+
```

```
+------------------------------------------+
|                游戏界面                   |
+------------------------------------------+
| 当前阶段: [投票] - 剩余时间: 00:30        |
+------------------------------------------+
|                                          |
|                                          |
|                                          |
|              [游戏主内容区]               |
|                                          |
|                                          |
|                                          |
+------------------------------------------+
|              任务状态板                   |
+------------------------------------------+
|  ●  ●  ○  ○  ○  | 拒绝: 1/5 | 时间: 10:45 |
|  成功 失败 进行中                          |
+------------------------------------------+
```

```
+------------------------------------------+
|             投票历史详情                  |
+------------------------------------------+
| 任务2 - 第1次投票 (拒绝)                  |
| 队长: 玩家3                              |
| 队员: 玩家3, 玩家5, 玩家7                |
| 结果: 3赞成 / 5反对                      |
|                                          |
| 赞成: 玩家3, 玩家5, 玩家7                |
| 反对: 玩家1, 玩家2, 玩家4, 玩家6, 玩家8   |
|                                          |
| 任务2 - 第2次投票 (通过)                  |
| 队长: 玩家4                              |
| 队员: 玩家1, 玩家4, 玩家8                |
| 结果: 5赞成 / 3反对                      |
|                                          |
| [返回]                    [关闭]          |
+------------------------------------------+
```

```
+------------------------------------------+
|             重要通知                      |
+------------------------------------------+
|                                          |
|  ⚠️ 警告: 连续拒绝已达到4次               |
|  再拒绝1次将导致本轮任务自动失败!          |
|                                          |
|  [确认]                                  |
|                                          |
+------------------------------------------+
```

## 游戏状态和进度展示界面（文本描述）

游戏状态和进度展示界面是阿瓦隆游戏中的信息中心，为玩家提供全局视角的游戏进程信息，设计遵循清晰、直观和信息层次化的原则：

1. **顶部状态栏**：

   - 位于游戏界面顶部，始终可见
   - 左侧显示当前游戏阶段图标和文字（如"队长选择队员"）
   - 中央显示阶段倒计时（如适用）
   - 右侧显示简化的任务进度指示器（成功/失败任务数）

2. **任务进度板**：

   - 设计为中世纪风格的圆桌或石板样式
   - 显示 5 个任务位置，每个位置有明确的状态标识：
     - 未开始：灰色空位
     - 进行中：闪烁的金色边框
     - 成功：蓝色/金色宝石或徽章
     - 失败：红色/黑色宝石或徽章
   - 每个任务位置显示所需队员数量
   - 当前任务有特殊标记和队长标识
   - 点击任务可查看详细历史信息（如适用）

3. **玩家状态区**：

   - 以圆形排列展示所有玩家的头像和简要状态
   - 当前队长有皇冠图标
   - 当前队员有特殊边框或标记
   - 已投票/已执行任务的玩家有完成标记
   - 离线玩家显示灰色效果
   - 玩家状态图标使用直观的视觉语言

4. **投票历史**：

   - 显示连续拒绝次数的计数器（如"拒绝: 2/5"）
   - 接近危险值（4/5）时有警告效果
   - 提供查看详细投票历史的按钮或手势
   - 详细历史以弹出层或侧边栏形式展示

5. **通知系统**：

   - 重要通知以横幅形式短暂显示在屏幕中央
   - 一般通知显示在底部通知区域
   - 通知使用不同颜色区分类型：
     - 信息：蓝色
     - 警告：黄色
     - 错误：红色
     - 成功：绿色
   - 通知有优雅的出现和消失动画
   - 重要通知可能伴随声音提示

6. **视觉设计**：

   - 整体使用阿瓦隆世界观的中世纪风格
   - 状态区域使用半透明背景，不遮挡主要游戏内容
   - 重要信息使用对比色和动画效果突出
   - 图标和文字结合，确保信息清晰易懂
   - 支持折叠/展开详细信息，平衡信息量和界面简洁性

7. **交互设计**：
   - 支持点击/轻触查看详细信息
   - 状态变化有平滑的过渡动画
   - 重要状态变化有醒目的视觉反馈
   - 支持手势操作（如滑动查看历史）
   - 关键信息区域有轻微的呼吸动画，引导注意力

通过这些设计元素，游戏状态和进度展示界面能够在不干扰游戏主体验的前提下，为玩家提供清晰、直观的游戏进程信息，帮助玩家理解当前游戏状态并做出明智的决策。

## 验收标准

1. **功能完整性**

   - 准确显示当前游戏阶段和倒计时
   - 正确展示任务进度和状态
   - 实时更新玩家状态和角色
   - 清晰记录和展示投票历史
   - 及时提供重要游戏事件的通知

2. **界面体验**

   - 状态信息展示清晰直观
   - 重要信息有醒目的视觉提示
   - 状态变化有适当的动画效果
   - 界面设计符合游戏整体风格
   - 在不同尺寸的设备上显示正常

3. **交互流畅性**

   - 状态更新及时，无明显延迟
   - 动画效果流畅，不出现卡顿
   - 交互操作响应迅速
   - 信息层次清晰，重要信息优先展示

4. **技术性能**
   - 状态更新不影响游戏主体验
   - 在低端设备上保持良好性能
   - 网络延迟情况下有适当的加载提示
   - 断线重连后能恢复正确的状态显示

## 技术依赖

- PIXI.js 渲染引擎
- GSAP 动画库
- Socket.IO 客户端（实时通信）
- 游戏状态管理模块
- 通知管理系统

## 工作量估计

3 人天

## 相关文档

- [阿瓦隆游戏规则](../../../阿瓦隆游戏规则.md)
- [Story1.3 游戏进行界面 README](../README.md)
- [Story1.3 技术方案](../技术方案.md)
- [Task1.3.2 实现任务组队界面](./Task1.3.2_实现任务组队界面.md)
- [Task1.3.4 实现任务执行界面](./Task1.3.4_实现任务执行界面.md)
