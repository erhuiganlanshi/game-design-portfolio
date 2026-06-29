# Erhui - Data Structure Design Document

Version: 0.1  
Author: 二辉橄榄石 & ChatGPT  
Last Update: 2026-06-28  
Related Docs: `01_GDD.md`, `02_TDD.md`, `04_Behavior.md`, `05_AI.md`

---

# 1. 文档目的

本文档定义 Erhui 的核心数据结构。

目标：

- 统一 C# 类、枚举、JSON 存档、配置文件的命名。
- 避免 Codex 在不同模块中重复创造相似结构。
- 保证 Needs、Behavior、AI、Memory、Save、Config 之间的数据格式一致。
- 让项目从一开始就具备可扩展性。

本文档应作为 Codex 编写代码时的主要数据规范。

---

# 2. 数据设计原则

## 2.1 Runtime Data 和 Save Data 分离

运行时数据用于游戏运行。

存档数据用于 JSON 序列化。

不要直接把 MonoBehaviour 保存到 JSON。

推荐结构：

```text
Runtime Class
    ↓ Convert
Save Data Class
    ↓ Serialize
JSON
```

例如：

```text
CatRuntimeState
    ↓
CatSaveData
    ↓
save.json
```

---

## 2.2 Config Data 和 Runtime Data 分离

配置数据定义规则。

运行时数据记录当前状态。

例如：

```text
NeedsConfig
    定义 Hunger 每分钟增长多少

CatNeeds
    记录当前 Hunger 是多少
```

不要在 `CatNeeds` 里写死变化速度。

---

## 2.3 AI 只能读取上下文

AI 只读取 `CatDecisionContext`。

AI 不能直接修改：

- CatNeeds
- Memory
- SaveData
- BehaviorRuntimeState
- Unity Transform

---

## 2.4 所有数值必须可限制

所有 0~100 的需求值都必须 Clamp。

```csharp
value = Mathf.Clamp(value, 0f, 100f);
```

---

# 3. 命名规范

## 3.1 C# 命名

| 类型 | 命名方式 | 示例 |
|---|---|---|
| Class | PascalCase | CatNeeds |
| Enum | PascalCase | CatMood |
| Method | PascalCase | UpdateNeeds |
| Public Field | PascalCase | Hunger |
| Private Field | camelCase | currentAction |
| Constant | UPPER_CASE | MAX_NEED_VALUE |

## 3.2 JSON 命名

JSON 字段统一使用 camelCase。

示例：

```json
{
  "hunger": 72,
  "sleepiness": 30,
  "currentAction": "Walk"
}
```

## 3.3 Action 命名

ActionId 使用 PascalCase 字符串。

示例：

```text
Sleep
Eat
FollowMouse
BuryPoop
```

---

# 4. 核心枚举

## 4.1 ActionId

建议使用 enum 管理内部行为，但与 AI 和 JSON 交互时使用字符串。

```csharp
public enum ActionId
{
    Idle,
    Walk,
    Run,
    Sleep,
    Eat,
    Drink,
    Poop,
    BuryPoop,
    Sit,
    Stretch,
    Groom,
    WatchMouse,
    FollowMouse,
    Scratch,
    Hide,
    LookAround,
    Play,
    Meow,
    Roll
}
```

---

## 4.2 CatMood

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

---

## 4.3 BehaviorPriority

```csharp
public enum BehaviorPriority
{
    Low = 0,
    Medium = 1,
    High = 2,
    Critical = 3
}
```

---

## 4.4 BehaviorState

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

## 4.5 AIProviderType

```csharp
public enum AIProviderType
{
    Mock,
    RuleBased,
    LocalQwen
}
```

---

## 4.6 InteractionType

```csharp
public enum InteractionType
{
    Click,
    Pet,
    DragStart,
    DragEnd,
    Hover,
    Feed,
    RefillWater,
    PlaceFurniture
}
```

---

## 4.7 MemoryEventType

```csharp
public enum MemoryEventType
{
    Behavior,
    Need,
    Interaction,
    World,
    AI,
    System
}
```

---

# 5. CatNeeds

`CatNeeds` 记录猫当前的需求数值。

所有字段范围为 0~100。

```csharp
[System.Serializable]
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

    public void ClampAll()
    {
        Energy = ClampNeed(Energy);
        Hunger = ClampNeed(Hunger);
        Thirst = ClampNeed(Thirst);
        Sleepiness = ClampNeed(Sleepiness);
        Happiness = ClampNeed(Happiness);
        Cleanliness = ClampNeed(Cleanliness);
        Health = ClampNeed(Health);
        Curiosity = ClampNeed(Curiosity);
        Affection = ClampNeed(Affection);
        Stress = ClampNeed(Stress);
        Loneliness = ClampNeed(Loneliness);
        ToiletNeed = ClampNeed(ToiletNeed);
    }

    private float ClampNeed(float value)
    {
        return UnityEngine.Mathf.Clamp(value, 0f, 100f);
    }
}
```

---

# 6. NeedDelta

`NeedDelta` 用于描述一个行为对 Needs 的影响。

例如 Eat 完成后：

```text
Hunger -35
Happiness +5
Thirst +5
```

建议结构：

```csharp
[System.Serializable]
public class NeedDelta
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

应用方式：

```csharp
public void ApplyDelta(NeedDelta delta)
{
    Energy += delta.Energy;
    Hunger += delta.Hunger;
    Thirst += delta.Thirst;
    Sleepiness += delta.Sleepiness;
    Happiness += delta.Happiness;
    Cleanliness += delta.Cleanliness;
    Health += delta.Health;
    Curiosity += delta.Curiosity;
    Affection += delta.Affection;
    Stress += delta.Stress;
    Loneliness += delta.Loneliness;
    ToiletNeed += delta.ToiletNeed;

    ClampAll();
}
```

---

# 7. CatRuntimeState

运行时猫状态。

```csharp
public class CatRuntimeState
{
    public CatNeeds Needs;
    public CatMood Mood;
    public ActionId CurrentAction;
    public Vector2 Position;
    public bool IsDragging;
    public bool IsSleeping;
    public float SecondsSinceLastInteraction;
    public List<ActionId> LastActions;
}
```

说明：

- `CurrentAction` 表示当前执行行为。
- `LastActions` 用于避免行为重复。
- `SecondsSinceLastInteraction` 用于判断孤独、黏人、叫声等行为。
- `IsDragging` 用于阻止 AI 决策在拖动中触发。

---

# 8. BehaviorDefinition

行为静态定义。

推荐未来使用 ScriptableObject 或 JSON 配置。

```csharp
[System.Serializable]
public class BehaviorDefinition
{
    public ActionId ActionId;
    public BehaviorPriority Priority;
    public bool CanBeInterrupted;
    public float MinDuration;
    public float MaxDuration;
    public float Cooldown;
    public string AnimationName;
    public string AudioName;
    public NeedDelta TickEffect;
    public NeedDelta FinishEffect;
    public List<string> RequiredConditions;
    public List<ActionId> NextForcedActions;
}
```

字段说明：

| 字段 | 说明 |
|---|---|
| ActionId | 行为 ID |
| Priority | 行为优先级 |
| CanBeInterrupted | 是否可被打断 |
| MinDuration | 最短持续时间 |
| MaxDuration | 最长持续时间 |
| Cooldown | 行为冷却 |
| AnimationName | 对应动画名 |
| AudioName | 对应音效名 |
| TickEffect | 行为持续期间的影响 |
| FinishEffect | 行为完成后的影响 |
| RequiredConditions | 前置条件 |
| NextForcedActions | 行为完成后强制衔接的行为 |

---

# 9. BehaviorRuntimeState

行为运行时状态。

```csharp
public class BehaviorRuntimeState
{
    public ActionId ActionId;
    public BehaviorState State;
    public float StartTime;
    public float Duration;
    public float ElapsedTime;
    public bool IsInterrupted;
    public string InterruptReason;
}
```

说明：

- `StartTime` 使用 Unity `Time.time`。
- `Duration` 从 BehaviorDefinition 的 MinDuration 和 MaxDuration 中随机。
- `ElapsedTime` 用于判断行为是否结束。
- `InterruptReason` 用于 Debug。

---

# 10. ActionCooldownData

记录行为冷却。

```csharp
[System.Serializable]
public class ActionCooldownData
{
    public string ActionId;
    public float RemainingSeconds;
}
```

运行时可以用 Dictionary：

```csharp
Dictionary<ActionId, float> actionCooldowns;
```

保存时转成 List：

```csharp
List<ActionCooldownData> cooldowns;
```

原因：

Unity JsonUtility 默认不支持 Dictionary 序列化。

---

# 11. WorldState

`WorldState` 是 AI 和行为系统读取环境的统一结构。

```csharp
public class WorldState
{
    public string TimeOfDay;
    public bool IsNight;
    public Vector2 MousePosition;
    public float MouseSpeed;
    public bool MouseNearCat;
    public Vector2 CatPosition;
    public Rect ScreenBounds;

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

MVP 阶段可简化为：

```csharp
public class WorldState
{
    public bool IsNight;
    public Vector2 MousePosition;
    public float MouseSpeed;
    public bool MouseNearCat;
    public bool HasFood;
    public bool HasWater;
    public bool HasCatBed;
    public bool HasLitterBox;
    public bool HasToy;
    public float SecondsSinceLastInteraction;
}
```

---

# 12. CatMemoryEvent

```csharp
[System.Serializable]
public class CatMemoryEvent
{
    public string Id;
    public MemoryEventType EventType;
    public string Description;
    public string CreatedAt;
    public float Importance;
}
```

字段说明：

| 字段 | 说明 |
|---|---|
| Id | 唯一 ID |
| EventType | 事件类型 |
| Description | 人类可读描述 |
| CreatedAt | ISO 时间字符串 |
| Importance | 重要程度，0~1 |

示例：

```json
{
  "id": "mem_0001",
  "eventType": "Interaction",
  "description": "主人摸了猫，猫看起来很开心。",
  "createdAt": "2026-06-28T14:30:00+08:00",
  "importance": 0.6
}
```

---

# 13. CatDecisionContext

发送给 AI 的上下文。

```csharp
public class CatDecisionContext
{
    public CatNeeds Needs;
    public CatMood Mood;
    public ActionId CurrentAction;
    public List<ActionId> LastActions;
    public WorldState World;
    public List<CatMemoryEvent> RecentMemory;
    public List<ActionId> AllowedActions;
    public List<ActionCooldownData> Cooldowns;
    public string PersonalityPrompt;
}
```

注意：

- 这是只读上下文。
- AI 不应持有对真实 CatNeeds 的引用。
- 构建 Context 时应复制数据，避免被异步修改影响。

---

# 14. CatDecision

AI 返回的标准决策。

```csharp
[System.Serializable]
public class CatDecision
{
    public string Action;
    public string Mood;
    public string Reason;
    public float Confidence;
    public string Provider;
}
```

示例：

```json
{
  "action": "FollowMouse",
  "mood": "Curious",
  "reason": "鼠标在附近移动，小猫很好奇。",
  "confidence": 0.78,
  "provider": "LocalQwen"
}
```

Unity 收到后必须转换为内部类型：

```csharp
ActionId parsedAction;
CatMood parsedMood;
```

转换失败则进入 fallback。

---

# 15. ValidatorResult

行为校验结果。

```csharp
public class ValidatorResult
{
    public bool IsValid;
    public ActionId FinalAction;
    public string RejectReason;
    public bool UsedFallback;
}
```

示例：

```json
{
  "isValid": false,
  "finalAction": "LookAround",
  "rejectReason": "Eat is on cooldown.",
  "usedFallback": true
}
```

---

# 16. FurnitureState

家具状态。

```csharp
[System.Serializable]
public class FurnitureState
{
    public string FurnitureId;
    public string FurnitureType;
    public float PositionX;
    public float PositionY;
    public bool IsActive;

    public float FoodAmount;
    public float WaterAmount;
    public float Durability;
}
```

说明：

- 食盆使用 `FoodAmount`。
- 水盆使用 `WaterAmount`。
- 猫抓板等家具可使用 `Durability`。
- 不相关字段可以保持默认值。

家具类型建议：

```text
FoodBowl
WaterBowl
CatBed
Toy
LitterBox
ScratchBoard
```

---

# 17. PlayerInteractionEvent

玩家交互事件。

```csharp
public class PlayerInteractionEvent
{
    public InteractionType Type;
    public Vector2 Position;
    public float Time;
    public string Description;
}
```

交互事件通常不长期保存，只用于短期 Memory 和状态调整。

---

# 18. SaveData 总结构

存档根结构。

```csharp
[System.Serializable]
public class GameSaveData
{
    public string Version;
    public CatSaveData Cat;
    public List<CatMemoryEvent> Memory;
    public List<ActionCooldownData> Cooldowns;
    public List<FurnitureState> Furniture;
    public GameSettingsData Settings;
    public string SavedAt;
}
```

---

# 19. CatSaveData

```csharp
[System.Serializable]
public class CatSaveData
{
    public CatNeeds Needs;
    public string Mood;
    public float PositionX;
    public float PositionY;
    public string CurrentAction;
    public List<string> LastActions;
}
```

说明：

- SaveData 中 Action 和 Mood 建议保存为 string。
- 读取时再解析 enum。
- 这样未来新增枚举时更容易做兼容。

---

# 20. GameSettingsData

```csharp
[System.Serializable]
public class GameSettingsData
{
    public bool EnableAI;
    public string AIProvider;
    public bool EnableSound;
    public float MasterVolume;
    public bool EnableDesktopMode;
    public bool EnableDebugPanel;
    public string Language;
}
```

默认设置：

```json
{
  "enableAI": true,
  "aiProvider": "RuleBased",
  "enableSound": true,
  "masterVolume": 0.8,
  "enableDesktopMode": false,
  "enableDebugPanel": true,
  "language": "zh-CN"
}
```

---

# 21. save.json 示例

MVP 阶段存档示例：

```json
{
  "version": "0.1",
  "cat": {
    "needs": {
      "energy": 70,
      "hunger": 45,
      "thirst": 30,
      "sleepiness": 20,
      "happiness": 65,
      "cleanliness": 80,
      "health": 95,
      "curiosity": 60,
      "affection": 25,
      "stress": 10,
      "loneliness": 40,
      "toiletNeed": 35
    },
    "mood": "Curious",
    "positionX": 0.5,
    "positionY": -1.2,
    "currentAction": "Idle",
    "lastActions": ["Walk", "Sit", "LookAround"]
  },
  "memory": [
    {
      "id": "mem_0001",
      "eventType": "Interaction",
      "description": "主人摸了猫。",
      "createdAt": "2026-06-28T14:30:00+08:00",
      "importance": 0.6
    }
  ],
  "cooldowns": [
    {
      "actionId": "Eat",
      "remainingSeconds": 120
    }
  ],
  "furniture": [
    {
      "furnitureId": "food_bowl_001",
      "furnitureType": "FoodBowl",
      "positionX": -2.0,
      "positionY": -1.0,
      "isActive": true,
      "foodAmount": 80,
      "waterAmount": 0,
      "durability": 100
    }
  ],
  "settings": {
    "enableAI": true,
    "aiProvider": "RuleBased",
    "enableSound": true,
    "masterVolume": 0.8,
    "enableDesktopMode": false,
    "enableDebugPanel": true,
    "language": "zh-CN"
  },
  "savedAt": "2026-06-28T14:35:00+08:00"
}
```

---

# 22. 配置数据

## 22.1 NeedsConfig

```csharp
[CreateAssetMenu(menuName = "Erhui/NeedsConfig")]
public class NeedsConfig : ScriptableObject
{
    public CatNeeds InitialNeeds;
    public NeedDelta PerMinuteChange;
}
```

示例：

```text
Hunger +1.0 / min
Thirst +1.2 / min
Sleepiness +0.8 / min
Loneliness +0.4 / min
Cleanliness -0.2 / min
```

---

## 22.2 BehaviorConfig

```csharp
[CreateAssetMenu(menuName = "Erhui/BehaviorConfig")]
public class BehaviorConfig : ScriptableObject
{
    public List<BehaviorDefinition> Behaviors;
}
```

示例：

```json
{
  "actionId": "Sleep",
  "priority": "High",
  "canBeInterrupted": true,
  "minDuration": 30,
  "maxDuration": 120,
  "cooldown": 300,
  "animationName": "Sleep",
  "audioName": "Sleep_01"
}
```

---

## 22.3 AIConfig

```csharp
[CreateAssetMenu(menuName = "Erhui/AIConfig")]
public class AIConfig : ScriptableObject
{
    public bool EnableAI;
    public AIProviderType ProviderType;
    public string LocalEndpoint;
    public float DecisionInterval;
    public float TimeoutSeconds;
    public int MaxMemoryEvents;
    public List<ActionId> AllowedActions;
}
```

默认值建议：

```text
EnableAI: true
ProviderType: RuleBased
LocalEndpoint: http://localhost:8000/decision
DecisionInterval: 45
TimeoutSeconds: 5
MaxMemoryEvents: 8
```

---

## 22.4 MovementConfig

```csharp
[CreateAssetMenu(menuName = "Erhui/MovementConfig")]
public class MovementConfig : ScriptableObject
{
    public float WalkSpeed;
    public float RunSpeed;
    public float ArriveDistance;
    public float RandomMoveRadius;
}
```

默认值建议：

```text
WalkSpeed: 1.2
RunSpeed: 2.5
ArriveDistance: 0.1
RandomMoveRadius: 3.0
```

---

## 22.5 MemoryConfig

```csharp
[CreateAssetMenu(menuName = "Erhui/MemoryConfig")]
public class MemoryConfig : ScriptableObject
{
    public int MaxMemoryCount;
    public int MaxPromptMemoryCount;
    public float ImportantMemoryThreshold;
}
```

默认值建议：

```text
MaxMemoryCount: 30
MaxPromptMemoryCount: 8
ImportantMemoryThreshold: 0.7
```

---

# 23. 数据转换规则

## 23.1 Runtime → Save

保存时：

```text
CatRuntimeState
    ↓
CatSaveData

MemorySystem
    ↓
List<CatMemoryEvent>

BehaviorManager
    ↓
List<ActionCooldownData>

FurnitureManager
    ↓
List<FurnitureState>
```

---

## 23.2 Save → Runtime

读取时：

```text
GameSaveData
    ↓
Validate Version
    ↓
CatRuntimeState
    ↓
Initialize Systems
```

读取失败时：

```text
Use Default Config
Create New Save
```

---

# 24. 版本迁移

存档必须包含 `version`。

当前版本：

```text
0.1
```

未来如果字段变化，需要写 Migration。

示例：

```csharp
public interface ISaveMigration
{
    string FromVersion { get; }
    string ToVersion { get; }
    GameSaveData Migrate(GameSaveData oldData);
}
```

MVP 阶段可以先只做：

```text
如果 version 不存在或无法识别，则备份旧存档并创建新存档。
```

---

# 25. JsonUtility 注意事项

Unity 自带 `JsonUtility` 有限制：

- 不支持 Dictionary。
- 不支持顶层数组。
- 对复杂泛型支持有限。
- 对 enum 通常可序列化，但为了兼容，存档建议用 string。

因此建议：

- 存档中 Dictionary 转 List。
- SaveData 尽量使用简单字段。
- 复杂配置可暂时用 ScriptableObject。
- 未来需要更灵活时再引入 Newtonsoft Json。

---

# 26. 数据合法性检查

读取存档和配置后必须进行检查。

## 26.1 Needs 检查

```text
所有 Needs Clamp 到 0~100。
Health 如果小于 1，则设为 1。
缺失字段使用默认值。
```

## 26.2 Mood 检查

```text
无法解析 Mood → Neutral
```

## 26.3 Action 检查

```text
无法解析 CurrentAction → Idle
LastActions 中未知 Action 直接移除
```

## 26.4 Furniture 检查

```text
FurnitureId 为空 → 生成新 ID
Position 缺失 → 使用默认位置
FoodAmount / WaterAmount Clamp 到 0~100
```

## 26.5 Cooldown 检查

```text
RemainingSeconds < 0 → 设为 0
未知 Action → 删除
```

---

# 27. Debug 输出结构

建议 Debug 面板直接读取以下结构：

```csharp
public class DebugSnapshot
{
    public CatNeeds Needs;
    public CatMood Mood;
    public ActionId CurrentAction;
    public List<ActionId> LastActions;
    public List<CatMemoryEvent> RecentMemory;
    public List<ActionCooldownData> Cooldowns;
    public CatDecision LastDecision;
    public ValidatorResult LastValidatorResult;
}
```

---

# 28. Codex 实现顺序

建议 Codex 按下面顺序实现数据结构：

```text
1. 创建 Enums
2. 创建 CatNeeds
3. 创建 NeedDelta
4. 创建 CatRuntimeState
5. 创建 BehaviorDefinition
6. 创建 BehaviorRuntimeState
7. 创建 WorldState
8. 创建 CatMemoryEvent
9. 创建 CatDecisionContext
10. 创建 CatDecision
11. 创建 ValidatorResult
12. 创建 FurnitureState
13. 创建 SaveData 相关类
14. 创建 Config ScriptableObject
15. 创建 DebugSnapshot
16. 创建数据转换工具类
```

---

# 29. 建议目录

```text
Assets/Scripts/

├── Data/
│   ├── Enums.cs
│   ├── CatNeeds.cs
│   ├── NeedDelta.cs
│   ├── CatRuntimeState.cs
│   ├── WorldState.cs
│   ├── CatDecision.cs
│   ├── CatDecisionContext.cs
│   ├── CatMemoryEvent.cs
│   ├── BehaviorDefinition.cs
│   ├── BehaviorRuntimeState.cs
│   ├── ValidatorResult.cs
│   ├── FurnitureState.cs
│   └── DebugSnapshot.cs
│
├── Save/
│   ├── GameSaveData.cs
│   ├── CatSaveData.cs
│   ├── GameSettingsData.cs
│   └── SaveDataConverter.cs
│
└── Config/
    ├── NeedsConfig.cs
    ├── BehaviorConfig.cs
    ├── AIConfig.cs
    ├── MovementConfig.cs
    └── MemoryConfig.cs
```

---

# 30. 当前版本结论

DataStructure v0.1 的核心目标是让项目所有模块围绕同一套数据工作。

后续 Codex 写代码时，应优先遵守本文档中的结构。

如果实现中发现字段不足，应先更新本文档，再修改代码。
