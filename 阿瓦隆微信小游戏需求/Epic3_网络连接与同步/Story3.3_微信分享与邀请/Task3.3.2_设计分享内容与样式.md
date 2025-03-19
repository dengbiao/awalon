# Task 3.3.2: 设计分享内容与样式

## 描述

本任务负责设计阿瓦隆微信小游戏的分享内容和样式，包括分享卡片的视觉设计、分享文案的撰写以及分享预览图的生成方案。良好的分享内容设计将提高分享的点击率和转化率，增强游戏的病毒式传播能力。

## 目标

1. 设计符合游戏风格的分享卡片视觉样式
2. 编写吸引人的分享文案，提高点击率
3. 开发能根据不同场景自动生成的分享预览图
4. 确保分享内容符合微信平台规范和要求

## 前置条件

1. 游戏 UI 设计规范已确定
2. 游戏素材库(图标、背景等)已准备就绪
3. 微信分享功能接口可用
4. 基础分享逻辑框架已实现

## 实现步骤

### 1. 分享卡片样式设计

#### 设计规范

分享卡片需要符合以下设计规范：

- 尺寸：建议 500x400 像素，确保在各种设备上显示清晰
- 配色：使用游戏主色调，保持品牌一致性
- 元素：包含游戏 logo、房间信息、创建者信息和行动号召
- 字体：使用清晰易读的无衬线字体，确保在小尺寸下仍可辨认

#### 设计草图

![分享卡片设计草图](../../../assets/share_card_design.png)

### 2. 分享文案设计

针对不同场景设计不同的分享文案：

1. **房间邀请分享**：

   - `"[创建者昵称]邀请你加入阿瓦隆游戏!"`
   - `"快来加入[创建者昵称]的阿瓦隆游戏，一起揪出隐藏的坏人！"`

2. **游戏结果分享**：

   - `"我在阿瓦隆[获胜/失败]了！来挑战一局？"`
   - `"[好人/坏人]阵营获胜！快来阿瓦隆一较高下！"`

3. **普通游戏分享**：
   - `"阿瓦隆 - 谁是莫德雷德的爪牙？快来一起推理！"`
   - `"最好玩的桌游阿瓦隆已上线微信小游戏，快来体验！"`

### 3. 分享预览图生成方案

```javascript
// 简化的分享图生成函数
function generateShareImage(options) {
  const { roomInfo, shareType, theme } = options;
  const canvas = wx.createCanvas();
  const ctx = canvas.getContext("2d");

  // 设置画布尺寸
  canvas.width = 500;
  canvas.height = 400;

  // 根据主题选择背景颜色
  if (theme === "dark") {
    ctx.fillStyle = "#1a237e"; // 深蓝色
  } else {
    ctx.fillStyle = "#3f51b5"; // 浅蓝色
  }
  ctx.fillRect(0, 0, 500, 400);

  // 绘制标题
  ctx.fillStyle = "#ffffff";
  ctx.font = "bold 28px sans-serif";
  ctx.textAlign = "center";
  ctx.fillText("阿瓦隆", 250, 100);

  // 根据分享类型绘制不同内容
  switch (shareType) {
    case "room_invite":
      drawRoomInvite(ctx, roomInfo);
      break;
    case "game_result":
      drawGameResult(ctx, roomInfo);
      break;
    default:
      drawDefaultShare(ctx);
  }

  // 转换为图片并返回
  return new Promise((resolve, reject) => {
    wx.canvasToTempFilePath({
      canvas,
      success: (res) => resolve(res.tempFilePath),
      fail: (err) => reject(err),
    });
  });
}

// 绘制房间邀请信息
function drawRoomInvite(ctx, roomInfo) {
  ctx.font = "20px sans-serif";
  ctx.fillText(`房间号: ${roomInfo.roomCode}`, 250, 160);
  ctx.fillText(`创建者: ${roomInfo.creatorName}`, 250, 200);
  ctx.fillText(
    `${roomInfo.currentPlayers}/${roomInfo.maxPlayers} 玩家`,
    250,
    240
  );

  // 行动号召
  ctx.font = "bold 24px sans-serif";
  ctx.fillText("点击加入游戏!", 250, 320);
}
```

### 4. 主题样式配置

创建主题配置文件，便于后续调整分享样式：

```javascript
// share/themeConfig.js
export const shareThemes = {
  default: {
    backgroundColor: "#3f51b5",
    textColor: "#ffffff",
    accentColor: "#ff9800",
    fontFamily: "sans-serif",
  },
  dark: {
    backgroundColor: "#1a237e",
    textColor: "#ffffff",
    accentColor: "#ffab40",
    fontFamily: "sans-serif",
  },
  result_win: {
    backgroundColor: "#2e7d32",
    textColor: "#ffffff",
    accentColor: "#ffc107",
    fontFamily: "sans-serif",
  },
  result_lose: {
    backgroundColor: "#c62828",
    textColor: "#ffffff",
    accentColor: "#ffecb3",
    fontFamily: "sans-serif",
  },
};

export function getTheme(themeKey) {
  return shareThemes[themeKey] || shareThemes.default;
}
```

### 5. 分享模板管理器

```javascript
// share/templateManager.js
import { getTheme } from "./themeConfig";

class ShareTemplateManager {
  // 获取分享文案
  getShareTitle(type, data = {}) {
    const templates = {
      room_invite: [
        `${data.creatorName}邀请你加入阿瓦隆游戏!`,
        `快来加入${data.creatorName}的阿瓦隆游戏，一起揪出隐藏的坏人！`,
      ],
      game_result: [
        `我在阿瓦隆${data.isWin ? "获胜" : "失败"}了！来挑战一局？`,
        `${data.isGood ? "好人" : "坏人"}阵营获胜！快来阿瓦隆一较高下！`,
      ],
      default: [
        "阿瓦隆 - 谁是莫德雷德的爪牙？快来一起推理！",
        "最好玩的桌游阿瓦隆已上线微信小游戏，快来体验！",
      ],
    };

    // 随机选择一条模板，增加多样性
    const templateList = templates[type] || templates.default;
    const randomIndex = Math.floor(Math.random() * templateList.length);
    return templateList[randomIndex];
  }

  // 获取分享配置
  async getShareConfig(type, data = {}) {
    // 确定主题
    let themeKey = "default";
    if (type === "game_result") {
      themeKey = data.isWin ? "result_win" : "result_lose";
    } else if (type === "room_invite") {
      themeKey = "dark";
    }

    // 生成分享图
    const shareImage = await generateShareImage({
      roomInfo: data.roomInfo,
      shareType: type,
      theme: themeKey,
    });

    // 返回完整配置
    return {
      title: this.getShareTitle(type, data),
      imageUrl: shareImage,
      query: data.query || "",
    };
  }
}

export default new ShareTemplateManager();
```

## 输出成果

1. 分享卡片的视觉设计方案
2. 多场景分享文案模板
3. 分享预览图生成代码
4. 分享主题样式配置
5. 分享模板管理器模块

## 验收标准

1. 分享卡片视觉设计符合游戏风格，信息展示清晰完整
2. 提供至少 3 种不同场景的分享文案，每种场景至少 2 个变体
3. 实现基于 Canvas 的分享预览图自动生成功能
4. 分享内容在 Android 和 iOS 平台上显示一致
5. 文字大小、颜色和排版合理，确保在小尺寸下仍清晰可辨
6. 分享样式符合微信平台规范要求

## 注意事项

1. 分享图的大小控制在 500KB 以内，避免过大影响加载速度
2. 文案设计应注意避免违规内容（如赌博、暴力等敏感词）
3. 分享图片生成需考虑不同设备的性能差异，优化绘制过程
4. 微信对分享图的要求可能会更新，需关注官方文档的变化

## 相关文档

- [微信小游戏 Canvas API 文档](https://developers.weixin.qq.com/minigame/dev/api/render/canvas/Canvas.html)
- [微信小游戏分享 API 文档](https://developers.weixin.qq.com/minigame/dev/api/share/wx.shareAppMessage.html)
- [微信内容规范](https://developers.weixin.qq.com/community/operate/doc/000c4e1a1c09b0e96a1a56dac5a408)
- [阿瓦隆游戏 UI 设计规范](../../../设计文档/UI设计规范.md)
