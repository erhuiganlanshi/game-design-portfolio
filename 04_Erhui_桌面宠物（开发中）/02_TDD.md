# Erhui - Technical Design Document (TDD)

Version: 0.1  
Author: 二辉橄榄石 & ChatGPT  
Last Update: 2026-06-28  
Engine: Unity 6 LTS  
Language: C#  
Target Platform: Windows  
Render Type: 2D  

---

# 1. 技术目标

Erhui 采用模块化架构。

目标：

- 让猫能够作为独立实体运行。
- 让行为、需求、记忆、AI、存档互相解耦。
- 让 MVP 可以先使用 Mock AI，后续再切换到 Qwen3。
- 让美术资源可以从占位图平滑替换为 Sprite Sheet。
- 让 Codex 能根据文档持续生成和维护代码。

核心原则：

- 高内聚
- 低耦合
- 数据驱动
- 易扩展
- 可调试
- 不硬编码关键参数

---

# 2. 工程目录

建议 Unity 工程采用以下目录：

```text
Assets/

├── Art/
│   ├── Placeholder/
│   └── Sprites/
│
├── Audio/
│
├── Animations/
│   ├── Clips/
│   └── Controllers/
│
├── Prefabs/
│   ├── Cat/
│   ├── Furniture/
│   └── UI/
│
├── Scenes/
│   └── MainScene.unity
│
├── Config/
│   ├── Behaviors/
│   ├── Needs/
│   └── AI/
│
├── StreamingAssets/
│   └── Save/
│
├── Scripts/
│   ├── AI/
│   ├── Cat/
│   ├── Behavior/
│   ├── Needs/
│   ├── Memory/
│   ├── Movement/
│   ├── Animation/
│   ├── Interaction/
│   ├── World/
│   ├── Save/
│   ├── Config/
│   ├── UI/
│   ├── Managers/
│   └── Utils/
│
└── Docs/
```

---

# 3. Scene 结构

第一版只需要一个主场景：

```text
MainScene

├── Main Camera
├── GameManager
├── WorldManager
├── SaveManager
├── AIManager
├── UIManager
├── Cat
│   ├── SpriteRenderer
│   ├── Animator
│   ├── Rigidbody2D
│   ├── Collider2D
│   ├── CatController
│   ├── NeedsSystem
│   ├── MemorySystem
│   ├── BehaviorManager
│   ├── MovementController
│   ├── CatAnimationController
│   └── InteractionController
└── DebugCanvas
```

---

# 4. 核心模块

项目核心模块：

| 模块 | 职责 |
|---|---|
| Cat | 统一协调猫相关组件 |
| Needs | 管理需求数值 |
| Behavior | 管理行为选择、校验、执行 |
| Movement | 处理移动 |
| Animation | 处理动画播放 |
| Interaction | 处理点击、拖拽、抚摸 |
| Memory | 记录短期事件 |
| AI | 请求本地模型或 Mock 决策 |
| World | 读取环境状态 |
| Save | 读写 JSON 存档 |
| Config | 读取行为和数值配置 |
| UI | Debug 和设置界面 |

---

# 5. Cat 架构

```text
CatController
    ├── NeedsSystem
    ├── MemorySystem
    ├── BehaviorManager
    ├── MovementController
    ├── CatAnimationController
    └── InteractionController
```

`CatController` 只负责协调，不写具体业务细节。

禁止将所有逻辑写入 `CatController`，避免形成 God Object。

---

# 6. 主数据流

```text
GameManager Tick
    ↓
NeedsSystem.UpdateNeeds()
    ↓
WorldManager.BuildWorldState()
    ↓
BehaviorManager.RequestDecision()
    ↓
AIManager.GetDecision()
    ↓
BehaviorValidator.Validate()
    ↓
ActionExecutor.Execute()
    ↓
Movement / Animation / Audio
    ↓
MemorySystem.AddEvent()
    ↓
SaveManager.AutoSave()
```

---

# 7. 行为系统

行为系统由以下类组成：

```text
BehaviorManager
BehaviorDefinition
BehaviorRuntimeState
BehaviorValidator
ActionExecutor
ActionCooldownTracker
```

## 7.1 BehaviorManager

职责：

- 维护当前行为。
- 请求新决策。
- 切换行为。
- 管理行为开始、执行、结束、中断。

## 7.2 BehaviorValidator

职责：

- 检查 Action 是否存在。
- 检查冷却时间。
- 检查需求条件。
- 检查家具条件。
- 检查当前行为是否可打断。
- 检查是否连续重复。
- 不合法时返回 fallback 行为。

## 7.3 ActionExecutor

职责：

- 根据 Action 执行移动、动画、音效。
- 施加行为完成后的数值变化。
- 写入 Memory。
- 通知事件系统。

---

# 8. Needs System

Needs 统一使用 0 到 100 的浮点数。

建议结构：

```csharp
public class CatNeeds
{
    public float Energy;
    public float Hunger;
    public float Thirst;
    public float Sleepiness;
    public float Happiness;
    public float Cleanliness;
    public float Health;
    public float Curiosity;
    public float Affection;
    public float Stress;
    public float Loneliness;
    public float ToiletNeed;
}
```

需求每秒更新一次，所有变化速率从配置读取。

---

# 9. Mood System

Mood 可以由 Needs 和 Memory 综合计算，也可以由 AI 决策返回后更新。

建议枚举：

```csharp
public enum CatMood
{
    Neutral,
    Happy,
    Relaxed,
    Curious,
    Lonely,
    Sleepy,
    Excited,
    Angry,
    Scared,
    Bored
}
```

MVP 阶段可以采用规则计算：

```text
Sleepiness > 75 → Sleepy
Happiness > 75 → Happy
Loneliness > 65 → Lonely
Curiosity > 65 → Curious
Stress > 65 → Scared
否则 Neutral
```

---

# 10. Movement

`MovementController` 只负责移动，不负责行为决策。

接口示例：

```csharp
MoveTo(Vector2 target, float speed)
Stop()
FaceTo(Vector2 target)
SetRandomDestination()
IsArrived()
```

MVP 阶段可以不做复杂寻路，只做屏幕范围内的直线移动。

---

# 11. Animation

`CatAnimationController` 只负责动画。

接口示例：

```csharp
Play(string animationName)
SetMood(CatMood mood)
SetFacingDirection(Vector2 direction)
Stop()
```

MVP 阶段允许多个行为共用同一动画，例如 `FollowMouse` 和 `Run` 都使用 `Walk` 占位动画。

---

# 12. Interaction

`InteractionController` 负责玩家输入。

第一版支持：

- 鼠标点击
- 鼠标拖拽
- 鼠标悬停
- 抚摸判定

交互事件应写入 Memory，并影响 Needs：

```text
点击猫 → Affection +1
抚摸猫 → Happiness +3, Stress -2
频繁点击 → Stress +3
拖动猫 → Stress +2
```

---

# 13. World State

`WorldManager` 提供环境信息。

建议结构：

```csharp
public class WorldState
{
    public string TimeOfDay;
    public bool IsNight;
    public Vector2 MousePosition;
    public float MouseSpeed;
    public Vector2 CatPosition;
    public bool HasFoodBowl;
    public bool HasFood;
    public bool HasWaterBowl;
    public bool HasWater;
    public bool HasCatBed;
    public bool HasLitterBox;
    public bool HasToy;
    public float SecondsSinceLastInteraction;
}
```

WorldState 只读，不直接修改猫状态。

---

# 14. AI 模块

AI 模块采用接口隔离，保证以后可以切换不同实现。

```csharp
public interface IAIDecisionProvider
{
    Task<CatDecision> GetDecisionAsync(CatDecisionContext context);
}
```

实现类：

```text
MockDecisionProvider
LocalQwenDecisionProvider
RuleBasedDecisionProvider
```

MVP 阶段先实现 `MockDecisionProvider` 和 `RuleBasedDecisionProvider`，再接入 `LocalQwenDecisionProvider`。

---

# 15. Memory System

Memory 采用 FIFO 队列。

结构示例：

```csharp
public class CatMemoryEvent
{
    public string EventType;
    public string Description;
    public DateTime Time;
    public float Importance;
}
```

默认保存最近 20 条。

Memory 会提供给 AI，但不能无限增长。

---

# 16. Save System

存档使用 JSON。

默认路径：

```text
Application.persistentDataPath / save.json
```

保存内容：

```json
{
  "version": "0.1",
  "cat": {
    "position": { "x": 0, "y": 0 },
    "mood": "Neutral",
    "needs": {}
  },
  "memory": [],
  "cooldowns": {},
  "settings": {}
}
```

保存时机：

- 启动后读取
- 退出时保存
- 每 60 秒自动保存
- 重要事件后保存

---

# 17. Config System

所有可调参数都从配置读取。

第一版可以使用 ScriptableObject。

后续可以迁移到 JSON。

配置类型：

```text
NeedsConfig
BehaviorConfig
AIConfig
MovementConfig
DebugConfig
```

禁止把行为持续时间、冷却时间、需求变化速率写死在代码中。

---

# 18. Event System

建议建立轻量事件系统，减少模块直接引用。

事件示例：

```text
NeedChanged
MoodChanged
ActionStarted
ActionFinished
ActionInterrupted
MemoryAdded
SaveLoaded
SaveCompleted
PlayerClickedCat
PlayerDraggedCat
AIDecisionReceived
AIDecisionFailed
```

---

# 19. Debug 系统

开发阶段必须提供 Debug 面板。

功能：

- 查看所有 Needs。
- 修改 Hunger、Thirst、Sleepiness、ToiletNeed。
- 强制触发 Action。
- 查看当前 Action。
- 查看当前 Mood。
- 查看行为冷却。
- 查看 AI 原始输出。
- 查看 Validator 拒绝原因。
- 清空 Memory。
- 保存 / 读取存档。

---

# 20. 桌面窗口

桌面宠物需要透明窗口、置顶、穿透等能力。

MVP 阶段可以先在普通 Unity 窗口中完成行为循环。

桌面透明窗口作为单独里程碑实现，避免过早增加复杂度。

建议开发顺序：

```text
普通窗口跑通玩法
    ↓
固定分辨率透明背景
    ↓
无边框窗口
    ↓
置顶
    ↓
可选鼠标穿透
```

---

# 21. Coding Style

命名规范：

- 类名：PascalCase
- 方法名：PascalCase
- 变量名：camelCase
- 常量：UPPER_CASE
- 一个文件一个主要类

工程规范：

- 禁止 God Object。
- 禁止跨模块直接修改数据。
- 禁止在 Update 中直接请求 AI。
- 禁止每帧写存档。
- 禁止模型输出绕过 Validator 直接执行。

---

# 22. MVP 技术完成标准

v0.1 完成时应满足：

- Cat Prefab 可以运行。
- Needs 每秒自动变化。
- 行为系统可以执行 MVP 行为。
- 行为切换有冷却和校验。
- Mock AI 可以返回决策。
- Qwen AI 可以作为可选实现接入。
- Memory 可以记录事件。
- Save 可以保存并恢复。
- Debug 面板可查看核心状态。
