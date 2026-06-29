# Erhui - Behavior Design Document

Version: 0.1  
Author: 二辉橄榄石 & ChatGPT  
Last Update: 2026-06-28  
Related Docs: `01_GDD.md`, `02_TDD.md`

---

# 1. 文档目的

本文档定义 Erhui 中“小猫行为系统”的设计规范。

本项目中的猫不是等待玩家点击后才响应的 UI 宠物，而是一只拥有自身需求、情绪、作息和偏好的桌面生命。

行为系统需要实现以下目标：

- 让猫能够自主行动。
- 让猫的行为与需求、情绪、环境有关。
- 让猫的行为具有一定随机性，但不能失控。
- 让 Qwen3 只负责“选择下一步行为”，不直接控制 Unity 角色。
- 让每个行为都能被 Codex 按统一规则实现。

---

# 2. 行为系统总览

行为系统由以下部分组成：

```text
Needs System
    ↓
World State
    ↓
Memory System
    ↓
Decision System
    ↓
Behavior Validator
    ↓
Action Executor
    ↓
Movement / Animation / Audio
```

其中：

- `Needs System` 负责管理饥饿、困意、快乐、卫生等需求。
- `World State` 负责提供时间、鼠标位置、家具位置等环境信息。
- `Memory System` 负责记录最近事件。
- `Decision System` 负责生成候选行为。
- `Behavior Validator` 负责检查行为是否合法。
- `Action Executor` 负责执行行为。
- `Movement / Animation / Audio` 负责表现层。

---

# 3. 行为设计原则

## 3.1 行为必须可控

Qwen3 不能直接生成任意行为。

所有行为必须来自预设行为表。

允许输出的 Action 必须属于 `AllowedActions`。

如果模型返回未知行为，则回退为 `Idle` 或 `LookAround`。

---

## 3.2 行为必须可打断

部分高优先级行为可以打断低优先级行为。

例如：

- 睡眠可以被玩家拖动打断。
- 吃饭可以被严重受惊打断。
- 排泄一般不可打断。
- 喝水、舔毛、发呆可以被打断。

---

## 3.3 行为不能无限重复

系统必须避免猫一直执行同一个行为。

基础规则：

- 同一行为不能连续执行超过 2 次。
- `Poop` 行为至少间隔 30 分钟。
- `Sleep` 行为结束后至少 5 分钟内不能再次睡觉。
- `Eat` 行为结束后至少 3 分钟内不能再次吃饭。
- `FollowMouse` 行为结束后至少 1 分钟内不能再次追鼠标。
- 如果 AI 输出违反冷却规则，则由 `Behavior Validator` 自动替换为备用行为。

---

## 3.4 行为要有猫味

行为不只是功能性动作。

每个行为都应该体现“小猫生活感”。

例如：

- 吃饭前可以先闻一下。
- 睡觉前可以转圈。
- 拉屎后需要埋屎。
- 无聊时可能盯着鼠标。
- 开心时可能打滚。
- 孤独时可能靠近鼠标。
- 困但不想睡时可能趴着发呆。

---

# 4. 行为数据结构

建议所有行为使用统一结构定义。

```csharp
public class BehaviorDefinition
{
    public string ActionId;
    public BehaviorPriority Priority;
    public bool CanBeInterrupted;
    public float MinDuration;
    public float MaxDuration;
    public float Cooldown;
    public string AnimationName;
    public string AudioName;
    public List<string> RequiredConditions;
    public List<string> FinishEffects;
}
```

行为优先级：

```csharp
public enum BehaviorPriority
{
    Low = 0,
    Medium = 1,
    High = 2,
    Critical = 3
}
```

行为状态：

```csharp
public enum BehaviorState
{
    Ready,
    Running,
    Finished,
    Interrupted,
    Failed
}
```

---

# 5. AllowedActions

第一版允许 Qwen3 输出以下行为：

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

后续版本再扩展：

```text
Jump
ChaseWindow
BlockMouse
StealIcon
PushDesktopItem
SitOnTaskbar
SleepNearMouse
StareAtPlayer
```

---

# 6. 行为优先级总表

| Action | Priority | Can Interrupt Others | Can Be Interrupted | MVP |
|---|---:|---|---|---|
| Idle | Low | No | Yes | Yes |
| Walk | Low | No | Yes | Yes |
| Run | Medium | No | Yes | Yes |
| Sleep | High | Yes | Yes | Yes |
| Eat | High | Yes | Yes | Yes |
| Drink | High | Yes | Yes | Yes |
| Poop | Critical | Yes | No | Yes |
| BuryPoop | High | Yes | No | Yes |
| Sit | Low | No | Yes | Yes |
| Stretch | Low | No | Yes | Yes |
| Groom | Medium | No | Yes | Yes |
| WatchMouse | Low | No | Yes | Yes |
| FollowMouse | Medium | No | Yes | Yes |
| Scratch | Medium | No | Yes | Later |
| Hide | Medium | No | Yes | Later |
| LookAround | Low | No | Yes | Yes |
| Play | Medium | No | Yes | Yes |
| Meow | Low | No | Yes | Yes |
| Roll | Low | No | Yes | Later |

---

# 7. 行为详细设计

---

## 7.1 Idle

### 行为说明

猫没有明确目标时进入待机状态。

Idle 不是完全静止，而是轻微呼吸、眨眼、尾巴摆动。

### 触发条件

- 当前没有更高优先级需求。
- AI 决策失败。
- 行为校验失败后的安全回退。

### 持续时间

3~10 秒。

### 可打断

可以被任意行为打断。

### 动画

`Idle`

### 状态影响

无明显影响。

### 记忆

通常不记录。

---

## 7.2 Walk

### 行为说明

猫在屏幕上慢慢散步。

### 触发条件

- 精力 > 20。
- 困意 < 80。
- 当前没有紧急需求。
- 随机探索概率触发。

### 持续时间

5~20 秒。

### 可打断

可以被高优先级行为打断。

### 动画

`Walk`

### 状态影响

- Energy -1
- Curiosity -1
- Happiness +1

### 记忆

可选记录：

```text
猫在桌面上散步了一会儿。
```

---

## 7.3 Run

### 行为说明

猫快速跑动，表现为突然兴奋或受到刺激。

### 触发条件

满足以下任一条件：

- 情绪为 `Excited`
- 好奇心 > 70
- 鼠标快速移动
- 玩家刚刚点击或拖动过猫
- 随机跑酷事件触发

### 持续时间

3~8 秒。

### 可打断

可以被 Critical 行为打断。

### 动画

`Run`

### 状态影响

- Energy -3
- Happiness +2
- Sleepiness +1

### 记忆

```text
猫突然在屏幕上跑了起来。
```

---

## 7.4 Sleep

### 行为说明

猫进入睡眠状态。

睡觉时猫会停止移动，可以在猫窝、屏幕角落或鼠标附近睡觉。

### 触发条件

满足以下任一条件：

- Sleepiness > 75
- Energy < 25
- 当前时间为夜晚
- 长时间没有玩家互动
- Mood 为 `Sleepy`

### 持续时间

30 秒 ~ 10 分钟。

MVP 阶段建议压缩为 30~120 秒，方便测试。

### 可打断

可以被以下事件打断：

- 玩家拖动猫。
- 玩家连续点击猫。
- 严重需求变化。
- Debug 指令。

不可被低优先级行为打断。

### 动画

`Sleep`

### 状态影响

每 10 秒：

- Energy +5
- Sleepiness -8
- Hunger +1
- Thirst +1
- Happiness +1

### 结束条件

满足以下任一条件：

- Sleepiness < 25
- Energy > 80
- 被玩家打断
- 睡眠持续时间结束

### 记忆

```text
猫睡了一觉。
```

---

## 7.5 Eat

### 行为说明

猫前往食盆并吃饭。

### 触发条件

- Hunger > 65。
- 食盆存在。
- 食盆内有食物。
- 当前没有更高优先级行为。

### 持续时间

5~15 秒。

### 可打断

可以被玩家拖动或严重惊吓打断。

### 动画

`Eat`

### 状态影响

行为完成后：

- Hunger -35
- Happiness +5
- Thirst +5
- Cleanliness -1

### 失败处理

如果食盆不存在或没有食物：

- 转为 `Meow`
- 增加记忆：猫发现没有食物
- Stress +3

### 记忆

```text
猫吃了一些东西。
```

---

## 7.6 Drink

### 行为说明

猫前往水盆并喝水。

### 触发条件

- Thirst > 60。
- 水盆存在。
- 水盆有水。

### 持续时间

4~10 秒。

### 可打断

可以被高优先级行为打断。

### 动画

`Drink`

### 状态影响

行为完成后：

- Thirst -35
- Happiness +2
- Cleanliness -1

### 失败处理

如果水盆不存在或没水：

- 转为 `Meow`
- Stress +2

### 记忆

```text
猫喝了水。
```

---

## 7.7 Poop

### 行为说明

猫需要排泄。

这是 Critical 行为，优先级最高。

### 触发条件

满足以下任一条件：

- ToiletNeed > 80。
- Eat 行为完成后一段时间后触发。
- 长时间未排泄。

### 持续时间

5~12 秒。

### 可打断

不可打断。

### 动画

`Poop`

### 状态影响

完成后：

- ToiletNeed -80
- Cleanliness -5
- Stress -3

### 后续行为

完成后必须进入：

`BuryPoop`

### 特殊规则

- 不允许连续 Poop。
- Poop 冷却时间不低于 30 分钟。
- MVP 阶段可以使用测试参数缩短为 3~5 分钟。

### 记忆

```text
猫上了厕所。
```

---

## 7.8 BuryPoop

### 行为说明

猫排泄后进行埋屎动作。

### 触发条件

- `Poop` 完成后强制触发。

### 持续时间

3~8 秒。

### 可打断

不可打断。

### 动画

`BuryPoop`

### 状态影响

- Cleanliness -2
- Happiness +1

### 记忆

```text
猫把便便埋了起来。
```

---

## 7.9 Sit

### 行为说明

猫坐在原地休息。

### 触发条件

- 没有紧急需求。
- Energy 处于中等。
- Mood 为 `Relaxed`、`Bored` 或 `Curious`。
- AI 选择观察环境。

### 持续时间

5~30 秒。

### 可打断

可以被任意高优先级行为打断。

### 动画

`Sit`

### 状态影响

- Energy +1
- Sleepiness +1
- Stress -1

### 记忆

通常不记录。

---

## 7.10 Stretch

### 行为说明

猫伸懒腰。

常发生在睡醒后、坐久后或准备移动前。

### 触发条件

满足以下任一条件：

- Sleep 行为结束后。
- Sit 行为持续较久后。
- 随机过渡动作触发。

### 持续时间

2~5 秒。

### 可打断

可以被高优先级行为打断。

### 动画

`Stretch`

### 状态影响

- Happiness +1
- Energy +1

### 记忆

通常不记录。

---

## 7.11 Groom

### 行为说明

猫舔毛、舔爪子或清洁自己。

### 触发条件

满足以下任一条件：

- Cleanliness < 60。
- Sleep 结束后。
- Eat 结束后。
- Mood 为 `Relaxed`。

### 持续时间

5~20 秒。

### 可打断

可以被玩家拖动或高优先级行为打断。

### 动画

`Groom`

### 状态影响

- Cleanliness +8
- Stress -2
- Sleepiness +1

### 记忆

```text
猫认真舔了舔毛。
```

---

## 7.12 WatchMouse

### 行为说明

猫盯着鼠标看。

这是表现“猫有自己的注意力”的关键行为。

### 触发条件

满足以下任一条件：

- 鼠标静止超过 5 秒。
- 鼠标在猫附近。
- Curiosity > 50。
- Mood 为 `Curious`。

### 持续时间

3~15 秒。

### 可打断

可以被 FollowMouse、Eat、Sleep 等行为打断。

### 动画

`WatchMouse`

### 状态影响

- Curiosity -1
- Happiness +1

### 记忆

通常不记录。

---

## 7.13 FollowMouse

### 行为说明

猫追逐鼠标。

### 触发条件

满足以下条件：

- Curiosity > 60。
- Energy > 30。
- 鼠标移动速度 > 阈值。
- 鼠标距离猫不超过一定范围。

### 持续时间

3~12 秒。

### 可打断

可以被 Critical 或 High 行为打断。

### 动画

`Run` 或 `Walk`

### 状态影响

- Energy -4
- Happiness +3
- Curiosity -5
- Sleepiness +1

### 结束条件

满足以下任一条件：

- 追逐时间结束。
- 鼠标停止移动。
- 猫距离鼠标过远。
- Energy < 20。

### 冷却时间

至少 60 秒。

### 记忆

```text
猫追了一会儿鼠标。
```

---

## 7.14 Scratch

### 行为说明

猫抓挠桌面边缘、家具或空气。

### 触发条件

- Mood 为 `Bored`。
- Stress > 40。
- Curiosity > 50。
- 当前附近有可抓挠对象。

### 持续时间

3~8 秒。

### 可打断

可以被高优先级行为打断。

### 动画

`Scratch`

### 状态影响

- Stress -3
- Happiness +2
- Cleanliness -1

### 记忆

```text
猫抓了抓附近的东西。
```

---

## 7.15 Hide

### 行为说明

猫躲到屏幕边缘、窗口后方或角落。

### 触发条件

满足以下任一条件：

- Stress > 65。
- 玩家频繁点击。
- 被拖动后心情不好。
- Mood 为 `Scared` 或 `Angry`。

### 持续时间

10~60 秒。

### 可打断

可以被 Eat、Drink、Poop、Sleep 打断。

### 动画

`Hide`

### 状态影响

- Stress -5
- Loneliness +2

### 记忆

```text
猫躲起来了。
```

---

## 7.16 LookAround

### 行为说明

猫四处张望。

用于 Idle 的替代行为，让猫不显得死板。

### 触发条件

- 无紧急需求。
- AI 决策失败。
- 当前行为结束后的过渡阶段。

### 持续时间

2~6 秒。

### 可打断

可以被任意行为打断。

### 动画

`LookAround`

### 状态影响

- Curiosity -1

### 记忆

通常不记录。

---

## 7.17 Play

### 行为说明

猫与玩具互动。

### 触发条件

满足以下条件：

- 玩具存在。
- Energy > 40。
- Happiness < 80。
- Curiosity > 40。
- 当前没有高优先级需求。

### 持续时间

5~20 秒。

### 可打断

可以被 Eat、Drink、Sleep、Poop 打断。

### 动画

`Play`

### 状态影响

- Energy -5
- Happiness +8
- Curiosity -5
- Sleepiness +2

### 记忆

```text
猫玩了一会儿玩具。
```

---

## 7.18 Meow

### 行为说明

猫发出叫声。

用于表达需求、吸引注意或随机卖萌。

### 触发条件

满足以下任一条件：

- 食盆没食物。
- 水盆没水。
- Loneliness > 60。
- 玩家长时间未互动。
- 随机事件触发。

### 持续时间

1~3 秒。

### 可打断

可以被任意高优先级行为打断。

### 动画

`Meow`

### 音效

`Meow_01`

### 状态影响

- Loneliness -1
- Stress -1

### 记忆

通常不记录，除非因缺食物或缺水触发。

---

## 7.19 Roll

### 行为说明

猫开心时在原地打滚。

### 触发条件

- Happiness > 75。
- Stress < 30。
- Energy > 30。
- 玩家刚刚互动。
- Mood 为 `Happy` 或 `Excited`。

### 持续时间

3~8 秒。

### 可打断

可以被高优先级行为打断。

### 动画

`Roll`

### 状态影响

- Happiness +2
- Energy -2

### 记忆

```text
猫开心地打了个滚。
```

---

# 8. 需求与行为关系

| Need | 数值变化 | 倾向行为 |
|---|---:|---|
| Hunger 高 | > 65 | Eat / Meow |
| Thirst 高 | > 60 | Drink / Meow |
| Sleepiness 高 | > 75 | Sleep / Sit |
| Energy 低 | < 25 | Sleep / Sit |
| Happiness 低 | < 40 | Play / FollowMouse / Meow |
| Cleanliness 低 | < 60 | Groom |
| Stress 高 | > 65 | Hide / Groom / Sleep |
| Curiosity 高 | > 60 | Walk / WatchMouse / FollowMouse |
| Loneliness 高 | > 60 | Meow / ApproachMouse / SitNearMouse |

---

# 9. 情绪与行为关系

| Mood | 行为倾向 |
|---|---|
| Happy | Roll / Play / Walk / FollowMouse |
| Relaxed | Sit / Groom / Sleep |
| Curious | WatchMouse / Walk / FollowMouse / LookAround |
| Lonely | Meow / FollowMouse / SitNearMouse |
| Sleepy | Sleep / Sit / Stretch |
| Excited | Run / Play / FollowMouse |
| Angry | Hide / Scratch / IgnorePlayer |
| Scared | Hide / Run |
| Bored | Walk / Scratch / LookAround / Meow |

---

# 10. 行为选择规则

行为选择分为三层：

```text
Hard Rule
    ↓
AI Decision
    ↓
Random Modifier
```

## 10.1 Hard Rule

Hard Rule 永远优先于 AI。

例如：

```text
如果 Health < 20，则禁止 Play、Run、FollowMouse。
如果 Hunger > 90 且有食物，则优先 Eat。
如果 Thirst > 90 且有水，则优先 Drink。
如果 ToiletNeed > 90，则强制 Poop。
如果 Sleepiness > 95，则强制 Sleep。
```

---

## 10.2 AI Decision

在没有强制规则时，由 Qwen3 根据当前状态选择行为。

Qwen3 只能输出：

```json
{
  "action": "Sleep",
  "mood": "Sleepy",
  "reason": "猫已经有点困了，想找个地方睡一会儿。"
}
```

---

## 10.3 Random Modifier

为了避免行为过于机械，系统可在 AI 决策后加入轻微随机修正。

例如：

- AI 选择 `Walk`，有 20% 概率替换为 `LookAround`。
- AI 选择 `Idle`，有 15% 概率替换为 `Stretch`。
- AI 选择 `Sit`，有 10% 概率替换为 `Groom`。

随机修正不能覆盖 Critical 行为。

---

# 11. 行为校验规则

`Behavior Validator` 负责检查 AI 输出是否合法。

校验顺序：

```text
1. Action 是否存在
2. 当前行为是否可被打断
3. 目标行为是否处于冷却
4. 需求条件是否满足
5. 家具条件是否满足
6. 是否违反重复规则
7. 是否符合当前健康状态
8. 是否允许随机替换
```

如果校验失败：

优先回退：

```text
LookAround
Sit
Idle
```

---

# 12. 行为执行流程

```text
StartAction
    ↓
Set CurrentAction
    ↓
Move To Target if needed
    ↓
Play Animation
    ↓
Play Audio if needed
    ↓
Apply Tick Effects
    ↓
FinishAction
    ↓
Apply Finish Effects
    ↓
Add Memory
    ↓
Start Cooldown
```

---

# 13. 行为中断流程

行为被中断时：

```text
InterruptAction
    ↓
Stop Movement
    ↓
Stop Animation or Transition
    ↓
Apply Interrupt Effects
    ↓
Add Memory if needed
    ↓
Enter Recovery Action
```

默认恢复行为：

```text
LookAround
Idle
```

被玩家拖动后：

```text
LookAround
Meow
Hide
```

---

# 14. MVP 阶段必须实现的行为

第一阶段不追求丰富，而是要跑通完整循环。

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

MVP 可暂缓：

```text
Run
Stretch
Groom
Scratch
Hide
Play
Roll
```

---

# 15. Debug 建议

开发阶段需要 Debug 面板，支持：

- 强制设置 Hunger
- 强制设置 Thirst
- 强制设置 Sleepiness
- 强制设置 ToiletNeed
- 强制触发某个 Action
- 清空冷却
- 查看当前 Mood
- 查看当前 Memory
- 查看 AI 原始输出
- 查看 Validator 拒绝原因

---

# 16. Codex 实现建议

建议 Codex 按以下顺序实现：

```text
1. 定义 ActionId 枚举
2. 定义 BehaviorDefinition
3. 定义 BehaviorRuntimeState
4. 实现 BehaviorManager
5. 实现 BehaviorValidator
6. 实现 ActionExecutor
7. 接入 CatController
8. 接入 Animator
9. 接入 NeedsSystem
10. 接入 MemorySystem
11. 接入 AI Decision
```

---

# 17. 行为配置示例

未来可将行为数据放入 JSON 或 ScriptableObject。

示例：

```json
{
  "actionId": "Eat",
  "priority": "High",
  "canBeInterrupted": true,
  "minDuration": 5,
  "maxDuration": 15,
  "cooldown": 180,
  "animationName": "Eat",
  "audioName": "Eat_01",
  "requiredConditions": [
    "Hunger > 65",
    "FoodBowl.Exists == true",
    "FoodBowl.HasFood == true"
  ],
  "finishEffects": [
    "Hunger -35",
    "Happiness +5",
    "Thirst +5",
    "Cleanliness -1"
  ]
}
```

---

# 18. 行为扩展模板

新增行为时，必须按以下模板填写。

```markdown
## ActionName

### 行为说明

说明这个行为是什么。

### 触发条件

列出可触发条件。

### 持续时间

最短时间与最长时间。

### 可打断

是否可打断，被什么打断。

### 动画

对应 Animation Clip 名称。

### 音效

对应 Audio Clip 名称，可选。

### 状态影响

开始、持续、结束时对 Needs 的影响。

### 失败处理

条件不满足时如何处理。

### 记忆

是否写入 Memory。
```

---

# 19. 当前版本结论

Behavior v0.1 的目标不是一次性实现所有动作。

真正目标是建立稳定、可扩展的行为框架。

只要 MVP 阶段能实现：

```text
需求变化 → 行为决策 → 行为校验 → 行为执行 → 状态变化 → 存档
```

Erhui 就已经具备“像一只猫一样生活”的基础。
