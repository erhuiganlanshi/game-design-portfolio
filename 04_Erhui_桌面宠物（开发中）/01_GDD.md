# Erhui - Game Design Document (GDD)

Version: 0.1
Author: 二辉橄榄石 & ChatGPT
Last Update: 2026-06-28
Engine: Unity 6 LTS
Platform: Windows
Render Type: 2D
AI: Local Qwen3 Decision Model

---

# 1. 项目简介

## 1.1 项目名称

Erhui（暂定）

## 1.2 项目类型

AI 单机桌面宠物 / 桌面生命模拟器

## 1.3 一句话定位

它不是一个等你点击才动的桌面宠物，而是一只住在你电脑里的猫。

## 1.4 核心卖点

- 完全单机运行。
- 使用 Unity 6 LTS 开发。
- 2D 桌面透明窗口表现。
- 小猫拥有自己的需求、情绪、记忆和作息。
- 本地 Qwen3 只负责“下一步想做什么”的决策，不负责聊天。
- 玩家主要是观察、陪伴和轻度互动，而不是完全控制猫。

---

# 2. 设计理念

Erhui 的核心不是“智能对话”，而是“生命感”。

猫应该像一只真实的小动物：

- 它会自己走动。
- 它会饿、困、无聊、孤独。
- 它会吃饭、喝水、睡觉、排泄、舔毛。
- 它会对鼠标、时间、互动频率做出反应。
- 它有时候理玩家，有时候不理玩家。
- 它不一定总是执行最优行为，而是有一点猫的随性。

玩家不是命令猫的主人，而是与猫共享电脑桌面的室友。

---

# 3. 目标体验

玩家打开电脑后，看到猫在桌面上生活。

理想体验包括：

- 玩家没有操作时，猫也会自主行动。
- 猫的行为看起来不是随机播放动画，而是有原因。
- 玩家能通过观察猫的行为猜到它的状态。
- 玩家偶尔互动，猫会产生轻微反馈。
- 长时间使用后，玩家会觉得这只猫有自己的习惯。

---

# 4. 核心循环

```text
Needs 自动变化
    ↓
感知世界状态
    ↓
读取短期记忆
    ↓
选择下一步行为
    ↓
执行行为
    ↓
修改状态
    ↓
写入记忆
    ↓
等待下一次决策
```

这个循环不依赖玩家点击。玩家交互只是影响循环中的状态、情绪和记忆。

---

# 5. 核心玩法

## 5.1 观察

玩家可以观察猫在桌面上行动，包括散步、睡觉、发呆、追鼠标、吃饭、喝水、排泄等。

## 5.2 照顾

玩家可以放置或补充：

- 食盆
- 水盆
- 猫窝
- 玩具
- 便盆a

## 5.3 互动

第一版支持：

- 点击猫
- 抚摸猫
- 拖动猫
- 观察猫对鼠标的反应

后续可扩展：

- 喂食
- 更换家具
- 更换皮肤
- 增加玩具
- 设置猫的性格

---

# 6. 猫的数据

## 6.1 基础需求

所有数值范围均为 0 到 100。

| 属性        | 含义     |
| ----------- | -------- |
| Energy      | 精力     |
| Hunger      | 饥饿     |
| Thirst      | 口渴     |
| Sleepiness  | 困意     |
| Happiness   | 快乐     |
| Cleanliness | 卫生     |
| Health      | 健康     |
| Curiosity   | 好奇心   |
| Affection   | 亲密度   |
| Stress      | 压力     |
| Loneliness  | 孤独     |
| ToiletNeed  | 排泄需求 |

## 6.2 数值变化原则

数值随时间缓慢变化。

示例：

```text
Hunger +1 / min
Thirst +1.2 / min
Sleepiness +0.8 / min
Loneliness +0.4 / min
Cleanliness -0.2 / min
```

实际数值以配置文件为准，不写死在代码里。

---

# 7. 情绪系统

Mood 不是连续数值，而是当前状态标签。

第一版 Mood：

```text
Happy
Relaxed
Curious
Lonely
Sleepy
Excited
Angry
Scared
Bored
Neutral
```

Mood 影响行为倾向。

例如：

- Happy 更容易 Roll、Play、FollowMouse。
- Sleepy 更容易 Sleep、Sit。
- Curious 更容易 WatchMouse、Walk、FollowMouse。
- Lonely 更容易 Meow、靠近鼠标。
- Angry 更容易 Hide、Scratch、无视玩家。

---

# 8. 世界信息

猫可以感知以下环境信息：

- 当前时间
- 是否夜晚
- 鼠标位置
- 鼠标速度
- 猫当前位置
- 屏幕边界
- 食盆位置
- 水盆位置
- 猫窝位置
- 玩具位置
- 便盆位置
- 玩家最近是否点击猫
- 玩家最近是否拖动猫
- 电脑是否长时间无操作

---

# 9. 行为列表

第一版目标行为：

```text
Idle
Walk
Run
Sleep
Eat
Drink
Poop
BuryPoop
Sit
Stretch
Groom
WatchMouse
FollowMouse
Scratch
Hide
LookAround
Play
Meow
Roll
```

MVP 必须实现：

```text
Idle
Walk
Sleep
Eat
Drink
Poop
BuryPoop
Sit
LookAround
WatchMouse
FollowMouse
Meow
```

---

# 10. AI 决策定位

Qwen3 不负责：

- 动画播放
- 角色移动
- 碰撞检测
- 寻路
- 数值修改
- 存档
- 直接控制 Unity 对象

Qwen3 只负责回答：

> 在当前状态下，这只猫下一步最想做什么？

输出格式：

```json
{
  "action": "Sleep",
  "mood": "Sleepy",
  "reason": "猫已经很困了，想找个舒服的地方睡一会儿。"
}
```

Unity 收到 Action 后，由行为系统执行对应行为。

---

# 11. 记忆系统

猫会记录短期事件，例如：

- 今天吃过饭。
- 刚才主人摸了我。
- 刚才被主人拖动了。
- 主人很久没有理我。
- 刚刚追过鼠标。
- 食盆没有食物。
- 刚睡醒。

Memory 默认只保留最近若干条，避免上下文过长。

---

# 12. 美术规范

第一阶段全部使用占位图。

后续正式美术建议使用：

- PNG 序列帧
- Sprite Sheet
- Unity Animation Clip
- Animator Controller

Unity 不建议直接使用 GIF 作为主要动画资源。

---

# 13. 音效规划

第一版可选音效：

- Meow
- Eat
- Drink
- Sleep
- Purr
- Run
- Scratch
- Poop

MVP 阶段可以先不接音效，优先完成行为循环。

---

# 14. 存档内容

存档采用 JSON。

保存内容包括：

- 猫的需求数值
- 当前情绪
- 当前坐标
- 最近记忆
- 行为冷却
- 家具状态
- 游戏设置

---

# 15. MVP 范围

v0.1 的目标不是做完整游戏，而是跑通“生命循环”。

MVP 完成标准：

- 桌面透明窗口可运行。
- 猫可以显示在桌面上。
- 猫可以自主移动。
- Needs 自动变化。
- 行为系统可以执行至少 8 个行为。
- 本地 AI 或 Mock AI 可以输出决策。
- 行为校验系统可以阻止非法行为。
- JSON 存档可以保存和读取。
- Debug 面板可以查看核心数值。

---

# 16. 后续版本规划

## v0.2 行为扩展

- 增加舔毛、玩耍、躲藏、打滚。
- 增加行为冷却和重复限制。
- 增加更多过渡动画。

## v0.3 家具与环境

- 食盆、水盆、猫窝、玩具、便盆可放置。
- 家具影响行为选择。
- 增加屏幕角落和任务栏附近的特殊位置。

## v0.4 AI 记忆增强

- 增加长期偏好。
- 增加玩家使用习惯记录。
- 增加猫的性格差异。

## v1.0 发布版

- 完整美术替换。
- 稳定桌面透明窗口。
- 完整设置界面。
- 打包 Windows 单机版。
- 提供 No-AI 模式和 Local-AI 模式。
