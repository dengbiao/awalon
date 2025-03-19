# Task 2.1.1: 实现角色随机分配逻辑

## 描述

实现阿瓦隆游戏中的角色随机分配算法，根据游戏人数和角色配置，公平地将角色分配给每个玩家。

## 详细需求

### 功能需求

1. 支持 5-10 人的游戏角色分配
2. 根据游戏规则，确保每局游戏中好人和坏人的数量平衡
3. 支持必选角色设置（如梅林、刺客必选）
4. 支持可选角色配置（如是否启用莫甘娜、派西维尔等）
5. 确保角色分配的随机性和公平性
6. 支持自定义角色配置预设

### 技术要求

1. 实现角色配置管理模块，维护角色信息和配置规则
2. 实现角色分配算法，包括必选角色确定、随机填充和角色分配
3. 使用密码学安全的随机数生成器，确保分配公平
4. 在服务端执行分配逻辑，防止客户端作弊
5. 提供配置验证机制，确保角色配置合法有效

## 实现步骤

1. 设计角色类型和阵营数据结构
2. 实现不同人数下的标准角色配置规则
3. 实现角色配置验证逻辑
4. 实现角色池生成算法
5. 实现角色随机分配算法
6. 实现分配结果验证机制
7. 编写单元测试，验证各种情况下的分配结果

## 代码示例

### 角色配置逻辑

```typescript
// 根据玩家人数确定标准角色配置
function getStandardRoleConfig(playerCount: number): GameRoleConfig {
  switch (playerCount) {
    case 5:
      return {
        playerCount: 5,
        availableRoles: [
          RoleType.MERLIN,
          RoleType.LOYAL_SERVANT,
          RoleType.LOYAL_SERVANT,
          RoleType.ASSASSIN,
          RoleType.MINION,
        ],
        requiredRoles: [RoleType.MERLIN, RoleType.ASSASSIN],
        goodCount: 3,
        evilCount: 2,
      };
    case 6:
    // 6人配置
    // ... 其他人数配置
    default:
      throw new Error(`Unsupported player count: ${playerCount}`);
  }
}

// 验证角色配置合法性
function validateRoleConfig(config: GameRoleConfig): boolean {
  // 1. 验证总人数 = 好人 + 坏人
  // 2. 验证必选角色都在可用角色中
  // 3. 验证好人坏人比例合理
  // ...
}
```

### 角色分配算法

```typescript
function assignRoles(
  players: Player[],
  config: GameRoleConfig
): Map<string, PlayerRole> {
  // 1. 验证配置
  if (!validateRoleConfig(config)) {
    throw new Error("Invalid role configuration");
  }

  // 2. 准备角色池
  const rolePool = prepareRolePool(config);

  // 3. 随机打乱角色池
  shuffleArray(rolePool);

  // 4. 随机打乱玩家顺序
  const shuffledPlayers = [...players];
  shuffleArray(shuffledPlayers);

  // 5. 分配角色给玩家
  const playerRoles = new Map<string, PlayerRole>();
  shuffledPlayers.forEach((player, index) => {
    playerRoles.set(player.id, {
      playerId: player.id,
      roleType: rolePool[index],
      visiblePlayers: [], // 先初始化为空，后续会计算
    });
  });

  return playerRoles;
}

// 准备角色池
function prepareRolePool(config: GameRoleConfig): RoleType[] {
  const rolePool: RoleType[] = [];

  // 添加必选角色
  config.requiredRoles.forEach((role) => rolePool.push(role));

  // 计算还需要添加的好人和坏人数量
  let remainingGood = config.goodCount;
  let remainingEvil = config.evilCount;

  // 从必选角色中减去已分配的数量
  config.requiredRoles.forEach((role) => {
    if (isGoodRole(role)) remainingGood--;
    else remainingEvil--;
  });

  // 从可用角色中随机选择填充剩余角色
  const availableGoodRoles = config.availableRoles.filter(
    (r) => isGoodRole(r) && !config.requiredRoles.includes(r)
  );
  const availableEvilRoles = config.availableRoles.filter(
    (r) => !isGoodRole(r) && !config.requiredRoles.includes(r)
  );

  // 随机选择好人角色
  while (remainingGood > 0) {
    if (availableGoodRoles.length > 0) {
      const randomIndex = getSecureRandomInt(0, availableGoodRoles.length - 1);
      rolePool.push(availableGoodRoles[randomIndex]);
      availableGoodRoles.splice(randomIndex, 1);
    } else {
      rolePool.push(RoleType.LOYAL_SERVANT); // 默认添加忠臣
    }
    remainingGood--;
  }

  // 随机选择坏人角色
  while (remainingEvil > 0) {
    if (availableEvilRoles.length > 0) {
      const randomIndex = getSecureRandomInt(0, availableEvilRoles.length - 1);
      rolePool.push(availableEvilRoles[randomIndex]);
      availableEvilRoles.splice(randomIndex, 1);
    } else {
      rolePool.push(RoleType.MINION); // 默认添加爪牙
    }
    remainingEvil--;
  }

  return rolePool;
}

// 安全的随机数生成
function getSecureRandomInt(min: number, max: number): number {
  // 使用密码学安全的随机数生成
  const randomBuffer = new Uint32Array(1);
  crypto.getRandomValues(randomBuffer);
  const randomNumber = randomBuffer[0] / (0xffffffff + 1);
  return Math.floor(randomNumber * (max - min + 1)) + min;
}

// Fisher-Yates 洗牌算法
function shuffleArray<T>(array: T[]): void {
  for (let i = array.length - 1; i > 0; i--) {
    const j = getSecureRandomInt(0, i);
    [array[i], array[j]] = [array[j], array[i]];
  }
}
```

## 界面原型

虽然角色随机分配逻辑主要是后端功能，但在游戏设置界面中需要提供角色配置选项：

```
+------------------------------------------+
|           游戏设置 - 角色配置             |
+------------------------------------------+
|                                          |
|  玩家人数: [5] [6] [7] [8] [9] [10]      |
|                                          |
|  角色配置:                               |
|  [✓] 使用标准配置                        |
|  [ ] 自定义配置                          |
|                                          |
|  必选角色:                               |
|  [✓] 梅林                                |
|  [✓] 刺客                                |
|  [ ] 派西维尔                            |
|  [ ] 莫甘娜                              |
|  [ ] 莫德雷德                            |
|  [ ] 奥伯伦                              |
|                                          |
|  当前配置:                               |
|  好人阵营: 3人 (梅林, 忠臣x2)            |
|  坏人阵营: 2人 (刺客, 爪牙)              |
|                                          |
|  [重置默认]                 [确认配置]    |
+------------------------------------------+
```

```
+------------------------------------------+
|         角色分配结果 (仅管理员可见)       |
+------------------------------------------+
|                                          |
|  玩家1: 梅林 (好人阵营)                  |
|  玩家2: 刺客 (坏人阵营)                  |
|  玩家3: 忠臣 (好人阵营)                  |
|  玩家4: 忠臣 (好人阵营)                  |
|  玩家5: 爪牙 (坏人阵营)                  |
|                                          |
|  [重新分配]                 [开始游戏]    |
+------------------------------------------+
```

## 验收标准

1. 角色分配算法能够正确处理 5-10 人的游戏配置
2. 分配结果符合游戏规则要求的好人/坏人比例
3. 必选角色在每局游戏中都被正确分配
4. 分配过程具有足够的随机性，没有明显的模式
5. 配置验证能够正确识别无效的角色配置
6. 算法性能良好，能够快速完成分配

## 技术依赖

- TypeScript/JavaScript
- 安全随机数生成库
- 后端服务框架（NestJS 等）

## 工作量估计

1 人天

## 相关文档

- [阿瓦隆角色规则](待补充)
- [随机数安全指南](待补充)
